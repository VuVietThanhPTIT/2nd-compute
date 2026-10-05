# Luồng Host write qua dm multipath và iSCSI TCP

## Mục lục

- [Mở đầu nguyên văn](#mo-dau)
- [CHẶNG 1: KHỞI TẠO TẠI USER SPACE VÀ BÀN GIAO VÀO KERNEL](#chang-1)
  - [Bước 1.1: Chuẩn bị bộ đệm căn chỉnh tại User Space](#buoc-1-1)
  - [Bước 1.2: Thiết lập thanh ghi và phát lệnh SYSCALL](#buoc-1-2)
    - [ Chỉ lệnh \`syscall\`: Bẻ hướng thực thi và Đổi quyền Ring 3 → Ring 0](#chi-lenh-syscall)
  - [Bước 1.3: Dịch địa chỉ và Ghim trang RAM vật lý (pin\_user\_pages)](#buoc-1-3)
  - [Bước 1.4: Đóng gói cấu trúc struct bio](#buoc-1-4)
- [CHẶNG 2: BLOCK LAYER (blk-mq) & ĐỊNH TUYẾN MULTIPATH](#chang-2)
  - [Bước 2.1: Chuyển hóa từ bio thành struct request](#buoc-2-1)
  - [Bước 2.2: Xếp hàng tại blk-mq Software Staging Queue](#buoc-2-2)
  - [Bước 2.3: Phân giải đường đi tại Device Mapper (dm-multipath)](#buoc-2-3)
- [CHẶNG 3: SCSI SUBSYSTEM & ĐÓNG GÓI iSCSI PDU](#chang-3)
  - [Bước 3.1: Dịch Request thành SCSI Command (CDB)](#buoc-3-1)
  - [Bước 3.2: Khởi tạo struct scsi\_cmnd và Scatter-Gather List](#buoc-3-2)
  - [Bước 3.3: Đóng gói bản tin iSCSI (PDU Assembly)](#buoc-3-3)
- [CHẶNG 4: NETWORK STACK & ĐIỀU KHIỂN PHẦN CỨNG NIC (DMA PREP)](#chang-4)
  - [Bước 4.1: Bọc gói tin tại TCP/IP Stack](#buoc-4-1)
  - [Bước 4.2: DMA Mapping qua IOMMU (Intel VT-d / AMD-Vi)](#buoc-4-2)
  - [Bước 4.3: Điền vòng truyền (TX Ring Buffer)](#buoc-4-3)
  - [Bước 4.4: Kích hoạt chuông Doorbell qua PCIe MMIO](#buoc-4-4)
- [CHẶNG 5: VẬN CHUYỂN QUA FABRIC & XỬ LÝ TẠI STORAGE TARGET](#chang-5)
  - [Bước 5.1: Card mạng tự động rút dữ liệu bằng DMA (Scatter-Gather DMA)](#buoc-5-1)
  - [Bước 5.2: Mã hóa vật lý và Phát tín hiệu quang (Serialization)](#buoc-5-2)
  - [Bước 5.3: Tiếp nhận và Ghi bền vững tại Target](#buoc-5-3)
- [CHẶNG 6: CHIỀU HOÀN TẤT & PHỤC HỒI NGẮNG (COMPLETION PATH)](#chang-6)
  - [Bước 6.1: Target gửi phản hồi hoàn tất](#buoc-6-1)
  - [Bước 6.2: Card NIC Host tiếp nhận & Ghi nhận qua DMA](#buoc-6-2)
  - [Bước 6.3: Kích hoạt Ngắt phần cứng (Hardware Interrupt: MSI-X)](#buoc-6-3)
  - [Bước 6.4: Xử lý tầng mạng ngầm (SoftIRQ) và Khớp lệnh SCSI](#buoc-6-4)
  - [Bước 6.5: Dọn dẹp tài nguyên và Đánh thức Ứng dụng](#buoc-6-5)
- [BẢNG TỔNG KẾT ĐẶC TẢ SỰ BIẾN ĐỔI QUA 6 CHẶNG](#bang-tong-ket)
- [1. Nếu lưu luôn trên Local Disk (không qua mạng) thì khác ở đâu?](#giai-dap-1)
- [2. Dữ liệu chỉ nằm ở buffer đầu tiên, còn lại chỉ là con trỏ chỉ vào?](#giai-dap-2)
  - [Trường hợp A: Dùng O\_DIRECT (Cơ chế trong bài đo của bạn)](#truong-hop-a)
  - [Trường hợp B: Dùng Buffered I/O (Mặc định không cờ)](#truong-hop-b)
- [3. Đếm chính xác: Có mấy lần memcpy payload?](#giai-dap-3)
  - [Ngoại lệ: Khi nào số lần memcpy bị tăng lên?](#ngoai-le)
- [Kết luận quan trọng cho bài báo cáo của bạn](#ket-luan-goc)
- [Bổ sung các device logic và device ảo](#device-ao)
- [Bổ sung đối chiếu đường buffered I O](#buffered-io)
- [Bổ sung điều kiện khi đếm memcpy](#dieu-kien-memcpy)
- [Nguồn tham khảo cho nội dung bổ sung](#nguon-tham-khao)

<a id="mo-dau"></a>

## Mở đầu nguyên văn

<!-- ORIGINAL 0 BEGIN -->

Thiếu lời gọi hàm ở từng bước, kèm phần phân tích hàm(ý nghĩa, trình bày các tham số), thiếu sự nhắc tới các device ảo

<!-- ORIGINAL 0 END -->

<!-- ORIGINAL 2 BEGIN -->

Luồng Host-write với cờ O\_DIRECT qua dm-multipath và iscsi-tcp loại bỏ hoàn toàn tầng đệm trung gian (Page Cache), biến quá trình I/O thành một chuỗi các thao tác chuyển hóa dữ liệu/tín hiệu liên tục: từ địa chỉ ảo trong không gian tiến trình, qua các cấu trúc mô tả của nhân hệ điều hành, ánh xạ thành địa chỉ phần cứng, và cuối cùng được điều chế thành sóng điện/quang truyền trên dây cáp.

<!-- ORIGINAL 2 END -->

<!-- ORIGINAL 4 BEGIN -->

Dưới đây là vi phẫu toàn bộ hành trình qua 6 chặng cụ thể. + kết hợp đọc luồng ở có page\_cache nữa để hiểu rõ hơn

<!-- ORIGINAL 4 END -->

<!-- ORIGINAL 5 BEGIN -->

(Vi phân kĩ hơn ở các thẻ 9)

<!-- ORIGINAL 5 END -->

<!-- ORIGINAL 6 BEGIN -->

<a id="chang-1"></a>

## CHẶNG 1: KHỞI TẠO TẠI USER SPACE VÀ BÀN GIAO VÀO KERNEL

<!-- ORIGINAL 6 END -->

<!-- ORIGINAL 7 BEGIN -->

(Biến đổi: Vùng đệm người dùng → Chỉ lệnh phần cứng → Khóa trang RAM vật lý)

<!-- ORIGINAL 7 END -->

<!-- ORIGINAL 8-22 BEGIN -->

```text
[ Ứng dụng Host (fio) ]
  - Buffer 4096B đã căn chỉnh (Host VA)
  - Nạp thanh ghi: RAX=1, RDI=fd, RSI=VA, RDX=4096
       │
       ▼ (Chỉ lệnh SYSCALL)
[ CPU Core: Privilege Level Switch (Ring 3 -> Ring 0) ]
       │
       ▼ (Bỏ qua Page Cache vì O_DIRECT)
[ Memory Management (get_user_pages / pin_user_pages) ]
  - CPU MMU dịch Host VA -> Host PA (Page Table Walk)
  - Khóa chặt trang RAM vật lý (Tránh OS thu hồi/swap)
       │
       ▼ (Cấp phát Metadata)
[ struct bio & struct bio_vec ]
  - bio_vec[0].bv_page = Con trỏ trỏ tới Host PA
```

<!-- ORIGINAL 8-22 END -->

<!-- ORIGINAL 23 BEGIN -->

<a id="buoc-1-1"></a>

### Bước 1.1: Chuẩn bị bộ đệm căn chỉnh tại User Space

Một đoạn code C thực hiện thao tác ghi dữ liệu:

<!-- ORIGINAL 23 END -->

<!-- ORIGINAL 25-30 BEGIN -->

```c
char buffer[4096];
memset(buffer, 'A', sizeof(buffer));
write(fd, buffer, 4096);

```

<!-- ORIGINAL 25-30 END -->

<!-- ORIGINAL 31 BEGIN -->

**Phần cứng tham gia:** Bộ nhớ RAM (DRAM), CPU L1/L2/L3 Cache.

<!-- ORIGINAL 31 END -->

<!-- ORIGINAL 32 BEGIN -->

**Cơ chế & Dữ liệu:** Ứng dụng cấp phát 4096 bytes dữ liệu (ví dụ toàn ký tự A = 0x41). Vì dùng O\_DIRECT, bộ đệm bắt buộc phải được căn chỉnh (Memory Aligned) theo bội số kích thước block (thường là 4 KiB) bằng hàm posix\_memalign().

<!-- ORIGINAL 32 END -->

<!-- ORIGINAL 33 BEGIN -->

**Dạng dữ liệu:** Dữ liệu nằm tại Host Virtual Address (Host VA) (ví dụ: 0x7f9a1000).

<!-- ORIGINAL 33 END -->

<!-- ORIGINAL 34 BEGIN -->

Lưu ý: phải có hàm posix\_memalign để căn chỉnh buffer để tránhHiệu ứng "Tràn qua 2 trang vật lý" (Page Boundary Crossing) (liên quan đến việc storage DMA kéo dữ liệu về)

<!-- ORIGINAL 34 END -->

<!-- ORIGINAL 35 BEGIN -->

Khi mở file với cờ O\_DIRECT, Linux Kernel kiểm tra nghiêm ngặt 3 yếu tố sau trước khi xử lý (thường đối chiếu với logical\_block\_size của ổ đĩa, là 512B hoặc 4096B):

<!-- ORIGINAL 35 END -->

<!-- ORIGINAL 36 BEGIN -->

- Địa chỉ con trỏ vùng nhớ (Memory Buffer Address): Con trỏ buf phải chia hết cho block size.

<!-- ORIGINAL 36 END -->

<!-- ORIGINAL 37 BEGIN -->

- Kích thước truyền dữ liệu (Data Length / Count): Số byte cần đọc/ghi (count) phải là bội số của block size (ví dụ: được ghi 4096, 8192; cấm ghi 4095 hay 100 bytes).

<!-- ORIGINAL 37 END -->

<!-- ORIGINAL 38 BEGIN -->

- Vị trí con trỏ tệp tin (File Offset): file-\>f\_pos (hoặc tham số offset trong pwrite) phải bắt đầu tại vị trí chia hết cho block size.

<!-- ORIGINAL 38 END -->

<!-- ORIGINAL 39 BEGIN -->

Nếu vi phạm bất kỳ điều kiện nào trong ba điều trên, kernel sẽ xử lý theo một trong hai cách:

<!-- ORIGINAL 39 END -->

<!-- ORIGINAL 40 BEGIN -->

- Trả về mã lỗi -EINVAL ngay lập tức tại VFS.

<!-- ORIGINAL 40 END -->

<!-- ORIGINAL 41 BEGIN -->

- (Trên một số hệ thống cũ hoặc cờ tương thích): Kernel tự động tạo một vùng đệm trung gian căn chỉnh gọi là Bounce Buffer, dùng CPU memcpy payload sang đó rồi mới DMA. Điều này phá hủy hoàn toàn bản chất Zero-Copy và làm suy sụp hiệu năng I/O.

<!-- ORIGINAL 41 END -->

<!-- ORIGINAL 42 BEGIN -->

—-\> posix\_memalign() là bản hợp đồng giữa Ứng dụng và Phần cứng: Ứng dụng chủ động xếp hàng dữ liệu ngay ngắn theo đúng từng khung trang 4KB trên RAM, đổi lại Phần cứng (IOMMU, Card mạng, SSD) có thể cắm ống hút DMA kéo thẳng dữ liệu đi mà không cần CPU nhúng tay vào copy hay chỉnh sửa bất kỳ byte nào.   


<!-- ORIGINAL 42 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.1

```c
int fd = open("/dev/mapper/mpath0", O_WRONLY | O_DIRECT);
void *buf = NULL;
int rc = posix_memalign(&buf, 4096, 4096);
/* Chỉ dùng buf khi fd >= 0 và rc == 0. */
memset(buf, 'A', 4096);
ssize_t n = pwrite(fd, buf, 4096, 0);
/* Kiểm tra n và lỗi; chỉ free(buf) sau khi I/O hoàn tất. */
```

**Ý nghĩa và tham số:** `open(path, flags)` mở device, trả `fd` hoặc `-1`; `O_WRONLY` yêu cầu ghi và `O_DIRECT` yêu cầu direct I/O. `posix_memalign(memptr, alignment, size)` ghi địa chỉ vào `*memptr`; alignment phải là lũy thừa hai và bội số `sizeof(void *)`; size là số byte cấp phát. Hàm trả mã lỗi trực tiếp, không dùng quy tắc errno của open. `memset(buf, value, count)` khởi tạo payload. `pwrite(fd, buf, count, offset)` ghi count byte tại offset, không đổi file offset dùng chung; trả số byte đã ghi hoặc `-1`.

**Device và lưu ý:** Đây là ví dụ bổ sung ghi dữ liệu vào device, không phải lệnh nên chạy trên ổ đang dùng. Mảng `char buffer[4096]` trong bản gốc không bảo đảm căn chỉnh 4096 byte. Yêu cầu căn chỉnh phụ thuộc filesystem/device; có thể hỏi `statx(..., STATX_DIOALIGN, ...)` khi hỗ trợ. posix_memalign căn chỉnh địa chỉ ảo, không bảo đảm vùng lớn liên tục vật lý. [S1][S2]

<!-- ORIGINAL 43 BEGIN -->

<a id="buoc-1-2"></a>

### Bước 1.2: Thiết lập thanh ghi và phát lệnh SYSCALL

<!-- ORIGINAL 43 END -->

<!-- ORIGINAL 44 BEGIN -->

**Phần cứng tham gia:** Các thanh ghi đa năng của CPU x86-64, cờ đặc quyền CPU (CPL).

<!-- ORIGINAL 44 END -->

<!-- ORIGINAL 45 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 45 END -->

<!-- ORIGINAL 46 BEGIN -->

- Thư viện hệ thống nạp các thông số vào thanh ghi:

<!-- ORIGINAL 46 END -->

<!-- ORIGINAL 47 BEGIN -->

- RAX = 1 (Mã syscall sys\_write) hoặc mã của io\_submit/pwritev2.

<!-- ORIGINAL 47 END -->

<!-- ORIGINAL 48 BEGIN -->

- RDI = fd (File descriptor đại diện cho /dev/mapper/mpath0).

<!-- ORIGINAL 48 END -->

<!-- ORIGINAL 49 BEGIN -->

- RSI = 0x7f9a1000 (Con trỏ Host VA trỏ tới buffer).

<!-- ORIGINAL 49 END -->

<!-- ORIGINAL 50 BEGIN -->

- RDX = 4096 (Độ dài byte cần ghi).

<!-- ORIGINAL 50 END -->

<!-- ORIGINAL 51 BEGIN -->

- CPU thực thi chỉ lệnh assembly syscall. Phần cứng lưu con trỏ lệnh RIP vào RCX, lưu RFLAGS vào R11, tráo đổi ngăn xếp (User Stack → Kernel Stack), và chuyển Current Privilege Level từ Ring 3 sang Ring 0.

<!-- ORIGINAL 51 END -->

<!-- ORIGINAL 53 BEGIN -->

> Điểm mấu chốt: Thanh ghi CPU chỉ mang con trỏ địa chỉ 64-bit (\`RSI\`), hoàn toàn không chứa 4096 byte dữ liệu payload. Payload vẫn nằm cố định trên RAM của tiến trình.

<!-- ORIGINAL 53 END -->

<!-- ORIGINAL 55 BEGIN -->

---

<!-- ORIGINAL 55 END -->

<!-- ORIGINAL 57 BEGIN -->

<a id="chi-lenh-syscall"></a>

####  Chỉ lệnh \`syscall\`: Bẻ hướng thực thi và Đổi quyền Ring 3 → Ring 0

<!-- ORIGINAL 57 END -->

<!-- ORIGINAL 59 BEGIN -->

Khi instruction \`syscall\` được CPU x86-64 thực thi, phần cứng thực hiện chuỗi hành động nguyên tử sau:

<!-- ORIGINAL 59 END -->

<!-- ORIGINAL 61 BEGIN -->

1\. Lưu vết luồng User Space:

<!-- ORIGINAL 61 END -->

<!-- ORIGINAL 62 BEGIN -->

 Phần cứng lưu con trỏ lệnh tiếp theo (\`RIP\`) vào thanh ghi \`RCX\`.

<!-- ORIGINAL 62 END -->

<!-- ORIGINAL 63 BEGIN -->

 Lưu các cờ trạng thái CPU (\`RFLAGS\`) vào thanh ghi \`R11\`.

<!-- ORIGINAL 63 END -->

<!-- ORIGINAL 66 BEGIN -->

2\. Đổi mức đặc quyền CPU (Privilege Level Switch):

<!-- ORIGINAL 66 END -->

<!-- ORIGINAL 67 BEGIN -->

 CPU thay đổi thanh ghi điều khiển, chuyển Current Privilege Level (CPL) từ Ring 3 (User Space) sang Ring 0 (Kernel Space).

<!-- ORIGINAL 67 END -->

<!-- ORIGINAL 70 BEGIN -->

3\. Chuyển con trỏ ngăn xếp (Stack Switch):

<!-- ORIGINAL 70 END -->

<!-- ORIGINAL 71 BEGIN -->

 CPU không dùng tiếp User Stack (để ngăn chặn nguy cơ tràn bộ đệm hoặc tấn công leo thang đặc quyền). Nó nạp địa chỉ ngăn xếp nhân (Kernel Stack) riêng của tiến trình hiện tại từ cấu trúc TSS (\`tss.sp0\`).

<!-- ORIGINAL 71 END -->

<!-- ORIGINAL 74 BEGIN -->

4\. Nhảy vào Kernel Entry:

<!-- ORIGINAL 74 END -->

<!-- ORIGINAL 75 BEGIN -->

 CPU nạp giá trị từ thanh ghi chuyên biệt MSR \`IA32\_LSTAR\` (Model-Specific Register) vào thanh ghi \`RIP\`. Giá trị này trỏ thẳng tới hàm đón đầu của Kernel: \`entry\_SYSCALL\_64\`.

<!-- ORIGINAL 75 END -->

<!-- ORIGINAL 79 BEGIN -->

> Phân biệt bản chất: Đây thuần túy là Privilege Switch (Đổi quyền) trên cùng một tiến trình, KHÔNG PHẢI Context Switch. Tiến trình sở hữu CPU vẫn là tiến trình ứng dụng ban đầu, cấu trúc \`current\` trỏ tới cùng một \`struct task\_struct\`.

<!-- ORIGINAL 79 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.2

```c
write(fd, buf, count);
/* Entry x86-64 minh họa: entry_SYSCALL_64 → __x64_sys_write
 * → ksys_write → vfs_write; không phải C gọi trực tiếp assembly entry. */
```

**Ý nghĩa và tham số:** `fd` là chỉ số trong bảng descriptor của tiến trình; `buf` là địa chỉ ảo user space; `count` là số byte. Với syscall write của ABI Linux x86-64, RAX mang số 1, RDI/RSI/RDX mang ba tham số. Đây là ABI của write, không áp nguyên bộ thanh ghi này cho io_submit hoặc pwritev2. Đầu vào entry là trạng thái thanh ghi CPU; đầu ra syscall là số byte hoặc lỗi.

**Device và lưu ý:** SYSCALL x86-64 không tự đổi RSP qua TSS.sp0 như lời mô tả trong bản gốc: mã entry của kernel thực hiện chuyển sang kernel stack. Chuyển đặc quyền tự nó không phải context switch, nhưng syscall có thể ngủ và dẫn tới scheduler đổi tiến trình.

<!-- ORIGINAL 81 BEGIN -->

<a id="buoc-1-3"></a>

### Bước 1.3: Dịch địa chỉ và Ghim trang RAM vật lý (pin\_user\_pages)

<!-- ORIGINAL 81 END -->

<!-- ORIGINAL 82 BEGIN -->

**Phần cứng tham gia:** Bộ quản lý bộ nhớ trên CPU (MMU - Memory Management Unit), thanh ghi CR3.

<!-- ORIGINAL 82 END -->

<!-- ORIGINAL 83 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 83 END -->

<!-- ORIGINAL 84 BEGIN -->

- Do có cờ O\_DIRECT, VFS bỏ qua hoàn toàn việc cấp phát Page Cache và hàm copy\_from\_user(). Không có bất kỳ thao tác memcpy payload nào xảy ra.

<!-- ORIGINAL 84 END -->

<!-- ORIGINAL 85 BEGIN -->

- Kernel gọi iov\_iter\_get\_pages2() / pin\_user\_pages(). MMU của CPU thực hiện tra bảng phân trang (Page Table Walk) từ thanh ghi CR3 để dịch:

<!-- ORIGINAL 85 END -->

<!-- ORIGINAL 86 BEGIN -->

- Host VA  (0x7f9a1000) ----\> Host PA (0x1a2b3000)

<!-- ORIGINAL 86 END -->

<!-- ORIGINAL 87 BEGIN -->

- Kernel tăng biến đếm tham chiếu (page\_ref\_inc) của khung trang vật lý này để ghim (pin) nó lại, ngăn không cho hệ điều hành di dời trang hoặc swap ra đĩa trong lúc I/O đang diễn ra.  


<!-- ORIGINAL 87 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.3

```c
/* API ghim trang minh họa, không phải mọi đường direct I/O gọi trực tiếp hàm này. */
long nr = pin_user_pages_fast(start, nr_pages, gup_flags, pages);
```

**Ý nghĩa và tham số:** `start` là địa chỉ ảo user space; `nr_pages` là số trang yêu cầu; `gup_flags` quy định cách lấy trang; `pages` nhận mảng con trỏ `struct page *`. Kết quả là số trang đã ghim hoặc lỗi, cần xử lý cả kết quả thiếu trang. Khi ghi xuống storage, thiết bị đọc buffer nên không tự động đặt FOLL_WRITE; cờ đó dùng khi phía kernel/thiết bị cần sửa nội dung trang. Direct I/O thực tế có thể đi qua các helper iov_iter và bio để lấy/ghim trang.

**Device và lưu ý:** `struct page *` là con trỏ kernel tới metadata của trang, không phải số Host PA. MMU dịch địa chỉ trong quá trình truy cập; kernel quản lý trang qua cấu trúc và bảng trang. Chỉ unpin cho trang lấy theo cơ chế pin tương ứng. [S3]

<!-- ORIGINAL 88 BEGIN -->

<a id="buoc-1-4"></a>

### Bước 1.4: Đóng gói cấu trúc struct bio

<!-- ORIGINAL 88 END -->

<!-- ORIGINAL 89 BEGIN -->

**Phần cứng tham gia:** Bộ cấp phát bộ nhớ Kernel (slab / kmalloc).

<!-- ORIGINAL 89 END -->

<!-- ORIGINAL 90 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 90 END -->

<!-- ORIGINAL 91 BEGIN -->

- Kernel khởi tạo cấu trúc mô tả I/O mức khối: struct bio.

<!-- ORIGINAL 91 END -->

<!-- ORIGINAL 92 BEGIN -->

- Gắn mảng struct bio\_vec:

<!-- ORIGINAL 92 END -->

<!-- ORIGINAL 93 BEGIN -->

- bv\_page: Chứa con trỏ trỏ thẳng tới khung trang vật lý Host PA (0x1a2b3000).

<!-- ORIGINAL 93 END -->

<!-- ORIGINAL 94 BEGIN -->

- bv\_len: 4096.

<!-- ORIGINAL 94 END -->

<!-- ORIGINAL 95 BEGIN -->

- bv\_offset: 0.

<!-- ORIGINAL 95 END -->

<!-- ORIGINAL 96 BEGIN -->

- bi\_iter.bi\_sector: Lưu vị trí sector bắt đầu cần ghi trên thiết bị logic /dev/mapper/mpath0.

<!-- ORIGINAL 96 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.4

```c
/* Mã minh họa API bio, các biến đã được caller chuẩn bị. */
struct bio *bio = bio_alloc(bdev, nr_vecs, REQ_OP_WRITE, GFP_KERNEL);
int added = bio_add_page(bio, page, len, page_offset);
bio->bi_iter.bi_sector = sector;
```

**Ý nghĩa và tham số:** `bdev` là block device đích; `nr_vecs` là sức chứa vector; `REQ_OP_WRITE` chỉ thao tác ghi; `GFP_KERNEL` là chính sách cấp phát. `bio_add_page` thêm trang `page`, số byte `len`, vị trí trong trang `page_offset`; trả số byte thêm được. `bi_sector` dùng đơn vị sector 512 byte của block layer. Caller phải kiểm tra lỗi cấp phát, giới hạn và số byte được thêm.

**Device và lưu ý:** `bv_page` trỏ tới struct page, `bv_len` là độ dài, `bv_offset` là offset trong trang. Nếu ghi file thường, filesystem còn phải ánh xạ file offset sang block trước; không đồng nhất file offset, LBA và địa chỉ RAM. [S4]

<!-- ORIGINAL 97 BEGIN -->

<a id="chang-2"></a>

## CHẶNG 2: BLOCK LAYER (blk-mq) & ĐỊNH TUYẾN MULTIPATH

<!-- ORIGINAL 97 END -->

<!-- ORIGINAL 98 BEGIN -->

(Biến đổi: Block I/O → Hàng đợi phần mềm → Phân phối đường đi vật lý)

<!-- ORIGINAL 98 END -->

<!-- ORIGINAL 99-110 BEGIN -->

```text
[ struct bio ]
       │
       ▼ (Hàm submit_bio)
[ Linux Block Layer: blk-mq ]
  - Tạo struct request từ bio
  - Xếp vào Software Staging Queue (Gắn với CPU Core hiện tại)
       │
       ▼ (Chuyển giao sang Device Mapper)
[ dm-multipath (/dev/mapper/mpath0) ]
  - Tra cứu Path Selector (Thuật toán service-time / round-robin)
  - Chọn đường tối ưu: /dev/sda (thay vì /dev/sdb)
  - Clone request -> Đẩy vào request_queue của /dev/sda
```

<!-- ORIGINAL 99-110 END -->

<!-- ORIGINAL 111 BEGIN -->

<a id="buoc-2-1"></a>

### Bước 2.1: Chuyển hóa từ bio thành struct request

<!-- ORIGINAL 111 END -->

<!-- ORIGINAL 112 BEGIN -->

**Phần cứng tham gia:** CPU Cache, Bộ nhớ nhân Linux.

<!-- ORIGINAL 112 END -->

<!-- ORIGINAL 113 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 113 END -->

<!-- ORIGINAL 114 BEGIN -->

- bio được đẩy vào Block Layer qua hàm submit\_bio().

<!-- ORIGINAL 114 END -->

<!-- ORIGINAL 116 BEGIN -->

- blk-mq tiếp nhận, kiểm tra khả năng gom cụm (I/O Merging). Nếu không có lệnh liền kề, nó cấp phát một đối tượng struct request từ pool bộ nhớ đệm.

<!-- ORIGINAL 116 END -->

<!-- ORIGINAL 117 BEGIN -->

- struct request liên kết danh sách các bio. Payload vẫn nằm yên tại khung trang vật lý 0x1a2b3000, chỉ có các con trỏ quản lý được chuyển tiếp.

<!-- ORIGINAL 117 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.1

```c
submit_bio(bio);
/* Tầng blk-mq thường có blk_mq_submit_bio(bio). */
```

**Ý nghĩa và tham số:** `bio` mô tả device, thao tác, sector và các trang dữ liệu. submit_bio giao I/O cho block layer; hoàn tất được báo qua callback bio, không phải hàm trả ngay số byte đã ghi. blk-mq có thể gom nhiều bio vào request hoặc chia I/O theo giới hạn; không có quan hệ cố định một write bằng một bio bằng một request.

**Device và lưu ý:** Device Mapper có chế độ bio-based và request-based. Vị trí tạo request/clone phụ thuộc mode; sơ đồ bản gốc là một mô hình, không phải call stack bắt buộc. [S4]

<!-- ORIGINAL 118 BEGIN -->

<a id="buoc-2-2"></a>

### Bước 2.2: Xếp hàng tại blk-mq Software Staging Queue

<!-- ORIGINAL 118 END -->

<!-- ORIGINAL 119 BEGIN -->

**Phần cứng tham gia:** CPU Core cục bộ.

<!-- ORIGINAL 119 END -->

<!-- ORIGINAL 120 BEGIN -->

Cơ chế & Dữ liệu:


<!-- ORIGINAL 120 END -->

<!-- ORIGINAL 121 BEGIN -->

- Request được đưa vào hàng đợi phần mềm riêng biệt (ctx - Software Queue) gắn chặt với CPU Core đang xử lý.

<!-- ORIGINAL 121 END -->

<!-- ORIGINAL 122 BEGIN -->

- Cơ chế này đảm bảo tính chất Lockless nội bộ: CPU Core tự đẩy việc vào hàng đợi của mình mà không cần tranh chấp khóa vòng (Spinlock) với các CPU Core khác trong hệ thống.

<!-- ORIGINAL 122 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.2

```c
/* Giao diện callback dispatch của blk-mq. */
blk_status_t queue_rq(struct blk_mq_hw_ctx *hctx,
                      const struct blk_mq_queue_data *bd);
```

**Ý nghĩa và tham số:** `queue_rq` là callback trong blk_mq_ops, tên hàm triển khai phụ thuộc driver. `hctx` là hardware dispatch context; `bd->rq` trỏ tới request; `bd->last` thông báo request cuối của đợt dispatch. Kết quả là blk_status_t báo chấp nhận hoặc lỗi/tài nguyên thiếu. Software context thường gắn CPU, hctx là lớp ánh xạ tới queue của driver.

**Device và lưu ý:** Không phải mọi request đều dừng ở staging queue: có đường dispatch trực tiếp. blk-mq giảm tranh chấp nhưng không có nghĩa toàn đường I/O lockless. [S4]

<!-- ORIGINAL 123 BEGIN -->

<a id="buoc-2-3"></a>

### Bước 2.3: Phân giải đường đi tại Device Mapper (dm-multipath)

<!-- ORIGINAL 123 END -->

<!-- ORIGINAL 124 BEGIN -->

**Phần cứng tham gia:** Bảng định tuyến trong bộ nhớ Kernel.

<!-- ORIGINAL 124 END -->

<!-- ORIGINAL 125 BEGIN -->

- Cơ chế & Dữ liệu

<!-- ORIGINAL 125 END -->

<!-- ORIGINAL 126 BEGIN -->

- Thiết bị nhận lệnh là /dev/mapper/mpath0. Module dm\_multipath.ko can thiệp vào quá trình dispatch.

<!-- ORIGINAL 126 END -->

<!-- ORIGINAL 127 BEGIN -->

- Thuật toán chọn đường (Path Selector, ví dụ round-robin hoặc service-time dựa trên số lượng I/O đang nghẽn) phân tích trạng thái các đường dẫn:

<!-- ORIGINAL 127 END -->

<!-- ORIGINAL 128 BEGIN -->

- Path 1: Nối qua Card NIC 1 tới Target IP 1 (đại diện bằng /dev/sda).

<!-- ORIGINAL 128 END -->

<!-- ORIGINAL 129 BEGIN -->

- Path 2: Nối qua Card NIC 2 tới Target IP 2 (đại diện bằng /dev/sdb).

<!-- ORIGINAL 129 END -->

<!-- ORIGINAL 130 BEGIN -->

- Giả sử Path 1 được chọn: Device Mapper tạo một bản sao yêu cầu (Cloned Request) và chuyển hướng toàn bộ con trỏ dữ liệu sang hàng đợi phần cứng của block device /dev/sda.

<!-- ORIGINAL 130 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.3

```c
/* Callback Device Mapper request-based minh họa. */
int clone_and_map_rq(struct dm_target *ti, struct request *rq,
                     union map_info *map_context,
                     struct request **clone);
```

**Ý nghĩa và tham số:** `ti` chứa target mapping; `rq` là request gốc; `map_context` giữ dữ liệu riêng để dùng khi hoàn tất; `clone` nhận request được map xuống device đã chọn. Callback và path selector quyết định cách chuyển yêu cầu. Với mode bio-based, giao diện khác là `map(ti, bio)`, sửa mapping của bio.

**Device và lưu ý:** `/dev/mapper/mpath0` là block device logic do kernel Device Mapper tạo. `/dev/sda` và `/dev/sdb` ở ví dụ là hai path tới cùng LUN, không mặc nhiên là hai ổ vật lý độc lập. Không ghi riêng từng path trong khi đang dùng device multipath. [S5]

<!-- ORIGINAL 131 BEGIN -->

<a id="chang-3"></a>

## CHẶNG 3: SCSI SUBSYSTEM & ĐÓNG GÓI iSCSI PDU

<!-- ORIGINAL 131 END -->

<!-- ORIGINAL 132 BEGIN -->

(Biến đổi: Request → Lệnh đĩa chuẩn hóa SCSI → Gói tin giao thức iSCSI)

<!-- ORIGINAL 132 END -->

<!-- ORIGINAL 133-146 BEGIN -->

```text
[ Cloned Request tại /dev/sda ]
       │
       ▼ (Tầng SCSI Disk Driver - sd)
[ SCSI Command (CDB) ]
  - Khởi tạo mảng byte lệnh: 0x2A (WRITE_10) + LBA + Length
       │
       ▼ (Tầng SCSI Middle Layer - scsi_mod)
[ struct scsi_cmnd & Scatter-Gather List ]
  - SG-List: [Address: Host PA (0x1a2b3000), Length: 4096]
       │
       ▼ (Tầng iSCSI Driver - iscsi_tcp)
[ iSCSI Protocol Data Unit (PDU) ]
  - iSCSI Command Header (48 bytes: Opcode 0x01, LUN, Task Tag)
  - iSCSI Data Payload (Trỏ thẳng vào Host PA qua SG-List)
```

<!-- ORIGINAL 133-146 END -->

<!-- ORIGINAL 147 BEGIN -->

<a id="buoc-3-1"></a>

### Bước 3.1: Dịch Request thành SCSI Command (CDB)

<!-- ORIGINAL 147 END -->

<!-- ORIGINAL 148 BEGIN -->

**Phần cứng tham gia:** CPU ALU (tính toán dịch bit).

<!-- ORIGINAL 148 END -->

<!-- ORIGINAL 150 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 150 END -->

<!-- ORIGINAL 152 BEGIN -->

- Tầng đĩa SCSI (sd\_mod) dịch vị trí sector trên đĩa thành khung lệnh chuẩn SCSI CDB (Command Descriptor Block):

<!-- ORIGINAL 152 END -->

<!-- ORIGINAL 153 BEGIN -->

- Sử dụng lệnh WRITE\_10 (Mã Opcode 0x2A) hoặc WRITE\_16 (0x8A).

<!-- ORIGINAL 153 END -->

<!-- ORIGINAL 154 BEGIN -->

- Nạp LBA đích (Logical Block Address) và số lượng block (Transfer Length = 8 sectors).

<!-- ORIGINAL 154 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.1

```c
/* Symbol thường gặp trong SCSI disk driver, cần đối chiếu kernel đang chạy. */
blk_status_t sd_init_command(struct scsi_cmnd *cmd);
```

**Ý nghĩa và tham số:** `cmd` là lệnh SCSI gắn với request; driver điền CDB, hướng truyền và độ dài. Kết quả blk_status_t báo thành công hoặc lỗi chuẩn bị. WRITE(10)/WRITE(16) dùng LBA và số logical block của thiết bị SCSI; opcode khác với opcode iSCSI bao ngoài.

**Device và lưu ý:** 4096 byte tương ứng 8 logical block khi logical block size là 512 byte; nếu 4096 byte thì là 1 block. Không dùng cố định 8 sectors cho mọi CDB. [S6]

<!-- ORIGINAL 155 BEGIN -->

<a id="buoc-3-2"></a>

### Bước 3.2: Khởi tạo struct scsi\_cmnd và Scatter-Gather List

<!-- ORIGINAL 155 END -->

<!-- ORIGINAL 156 BEGIN -->

**Phần cứng tham gia:** Bộ nhớ Kernel.

<!-- ORIGINAL 156 END -->

<!-- ORIGINAL 157 BEGIN -->

Cơ chế & Dữ liệu:SCSI Mid-layer (scsi\_mod) bao bọc CDB vào cấu trúc struct scsi\_cmnd.

<!-- ORIGINAL 157 END -->

<!-- ORIGINAL 158 BEGIN -->

- Kernel chuyển đổi mảng bio\_vec thành danh sách phân tán bộ nhớ (Scatter-Gather List - SG list):

<!-- ORIGINAL 158 END -->

<!-- ORIGINAL 159 BEGIN -->

- sg\_dma\_address(sg\[0\]): Trỏ tới Host PA 0x1a2b3000.

<!-- ORIGINAL 159 END -->

<!-- ORIGINAL 160 BEGIN -->

sg\_dma\_len(sg\[0\]): 4096.

<!-- ORIGINAL 160 END -->

<!-- ORIGINAL 161 BEGIN -->

- SG list này chính là bản đồ chỉ đường cho phần cứng card mạng biết vị trí bốc dữ liệu sau này.

<!-- ORIGINAL 161 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.2

```c
int nr_sg = blk_rq_map_sg(queue, rq, sgl);
```

**Ý nghĩa và tham số:** `queue` là request_queue có giới hạn của device; `rq` là request; `sgl` là vùng scatterlist do caller chuẩn bị đủ sức chứa. Giá trị trả về là số phần tử SG sử dụng. Mỗi phần tử mô tả trang và đoạn byte; dữ liệu không nhất thiết liên tục vật lý.

**Device và lưu ý:** Trước DMA mapping, dùng thông tin trang/offset/length của SG. Chỉ xem sg_dma_address và sg_dma_len là địa chỉ/độ dài DMA sau DMA mapping phù hợp. Với iscsi-tcp software, NIC driver ánh xạ các fragment mạng ở tầng sau, không mặc nhiên dùng DMA mapping của SCSI HBA. [S7]

<!-- ORIGINAL 163 BEGIN -->

<a id="buoc-3-3"></a>

### Bước 3.3: Đóng gói bản tin iSCSI (PDU Assembly)

<!-- ORIGINAL 163 END -->

<!-- ORIGINAL 164 BEGIN -->

**Phần cứng tham gia:** Module libiscsi.ko và iscsi\_tcp.ko.

<!-- ORIGINAL 164 END -->

<!-- ORIGINAL 166 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 166 END -->

<!-- ORIGINAL 168 BEGIN -->

- Driver iscsi\_tcp tiếp nhận lệnh SCSI, khởi tạo cấu trúc iSCSI PDU (Basic Header Segment - BHS) kích thước 48 bytes:

<!-- ORIGINAL 168 END -->

<!-- ORIGINAL 170 BEGIN -->

- Byte 0: Opcode = 0x01 (SCSI Command).

<!-- ORIGINAL 170 END -->

<!-- ORIGINAL 172 BEGIN -->

- Byte 1: Cờ F (Final) và W (Write).

<!-- ORIGINAL 172 END -->

<!-- ORIGINAL 174 BEGIN -->

- Bytes 8-15: Logical Unit Number (LUN ID).

<!-- ORIGINAL 174 END -->

<!-- ORIGINAL 176 BEGIN -->

- Bytes 16-19: Initiator Task Tag (ITT - mã số định danh do Host tự gán để ghép cặp với phản hồi sau này).

<!-- ORIGINAL 176 END -->

<!-- ORIGINAL 178 BEGIN -->

- Bytes 20-23: Expected Data Transfer Length (4096).

<!-- ORIGINAL 178 END -->

<!-- ORIGINAL 180 BEGIN -->

- Dữ liệu payload 4 KiB không bị copy vào PDU Header; driver chỉ thiết lập cấu trúc con trỏ để socket TCP biết cần gửi 48 bytes Header trước, theo sau ngay là 4096 bytes dữ liệu từ SG list.

<!-- ORIGINAL 180 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.3

```c
/* Điểm vào libiscsi thường gặp, cần kiểm tra source của kernel lab. */
int iscsi_queuecommand(struct Scsi_Host *host,
                        struct scsi_cmnd *sc);
```

**Ý nghĩa và tham số:** `host` là SCSI host logic của session iSCSI; `sc` là lệnh có CDB và SG payload. Libiscsi cấp task, gán ITT, tạo command header và giao transport; kết quả phản ánh nhận lệnh hoặc cần retry, không phải xác nhận ghi bền vững.

**Device và lưu ý:** Payload có thể đi trong Immediate Data, Unsolicited Data-Out hoặc Data-Out sau R2T. Không phải luôn có một header 48 byte nối ngay toàn bộ 4096 byte. Phụ thuộc InitialR2T, ImmediateData, giới hạn segment và digest đã thương lượng. [S8]

<!-- ORIGINAL 182 BEGIN -->

<a id="chang-4"></a>

## CHẶNG 4: NETWORK STACK & ĐIỀU KHIỂN PHẦN CỨNG NIC (DMA PREP)

<!-- ORIGINAL 182 END -->

<!-- ORIGINAL 183 BEGIN -->

(Biến đổi: Gói tin lưu trữ → Gói tin mạng IP → Tín hiệu bus PCIe MMIO)

<!-- ORIGINAL 183 END -->

<!-- ORIGINAL 185-203 BEGIN -->

```text
[ iSCSI Header (48B) ] + [ SG-List trỏ Payload (4096B) ]
       │
       ▼ (Hàm kernel_sendpage / sock_sendmsg)
[ Linux TCP/IP Stack ]
  - Tạo struct sk_buff (skb)
  - Thêm TCP Header (Port 3260) + IP Header
  - Cập nhật TCP Sequence Number, Window Size
       │
       ▼ (Driver Card Mạng: ví dụ igb / ixgbe / mlx5_core)
[ DMA Mapping qua IOMMU ]
  - IOMMU dịch: Host PA (0x1a2b3000) -> IOVA (0x80001000)
       │
       ▼ (Nạp TX Descriptor vào RAM)
[ Ring Buffer của NIC (TX Ring) ]
  - TX Desc chứa con trỏ IOVA (0x80001000) và độ dài 4096B
       │
       ▼ (CPU ghi lệnh qua PCIe Bus)
[ Thanh ghi Doorbell MMIO trên NIC ]
  - CPU bắn PCIe TLP Memory Write báo có việc mới
```

<!-- ORIGINAL 185-203 END -->

<!-- ORIGINAL 204 BEGIN -->

<a id="buoc-4-1"></a>

### Bước 4.1: Bọc gói tin tại TCP/IP Stack

<!-- ORIGINAL 204 END -->

<!-- ORIGINAL 205 BEGIN -->

**Phần cứng tham gia:** CPU Core, cấu trúc mạng của Kernel.

<!-- ORIGINAL 205 END -->

<!-- ORIGINAL 206 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 206 END -->

<!-- ORIGINAL 207 BEGIN -->

- Giao vận iSCSI gọi socket interface. Kernel cấp phát cấu trúc struct sk\_buff (skb):  


<!-- ORIGINAL 207 END -->

<!-- ORIGINAL 208 BEGIN -->

- Phần đầu skb chứa: TCP Header (Source Port, Destination Port = 3260, Sequence Number) và IP Header (Source IP của Host, Dest IP của Target Portal).

<!-- ORIGINAL 208 END -->

<!-- ORIGINAL 210 BEGIN -->

- Payload của gói tin: Kernel sử dụng tính năng skb\_frag\_t (Zero-Copy Socket). Con trỏ trang RAM của skb được trỏ trực tiếp vào khung trang Host PA mà SG-list cung cấp. Tuyệt đối không dùng memcpy để nạp dữ liệu vào socket buffer.

<!-- ORIGINAL 210 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 4.1

```c
int sent = sock_sendmsg(sock, &msg);
/* Trên kernel phù hợp, msg.msg_flags có thể chứa MSG_SPLICE_PAGES. */
```

**Ý nghĩa và tham số:** `sock` là kernel socket TCP đã kết nối; `msg` là msghdr chứa iterator payload và cờ gửi. Kết quả là số byte nhận vào đường gửi hoặc mã lỗi; chưa chứng minh storage đã ghi xong. Các kernel cũ có đường sendpage; API và cách transport iscsi-tcp dùng chúng thay đổi theo phiên bản.

**Device và lưu ý:** Chỉ gắn tham chiếu trang khi iterator, socket và network path hỗ trợ. Copy/fallback vẫn có thể xảy ra, nên không suy ra zero-copy chỉ từ tên sock_sendmsg hoặc O_DIRECT. TCP phân đoạn/ghép gói độc lập với biên iSCSI PDU. [S9]

<!-- ORIGINAL 212 BEGIN -->

<a id="buoc-4-2"></a>

### Bước 4.2: DMA Mapping qua IOMMU (Intel VT-d / AMD-Vi)

<!-- ORIGINAL 212 END -->

<!-- ORIGINAL 213 BEGIN -->

**Phần cứng tham gia:** Chipset phần cứng IOMMU, PCIe Root Complex.

<!-- ORIGINAL 213 END -->

<!-- ORIGINAL 215 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 215 END -->

<!-- ORIGINAL 217 BEGIN -->

- Driver mạng gọi hàm Kernel DMA API: dma\_map\_single() hoặc dma\_map\_page().

<!-- ORIGINAL 217 END -->

<!-- ORIGINAL 219 BEGIN -->

- Thiết bị ngoại vi không thể đọc trực tiếp bằng Host PA nếu hệ thống bật IOMMU bảo vệ. IOMMU can thiệp, tạo một mục ánh xạ trong bảng trang I/O:

<!-- ORIGINAL 219 END -->

<!-- ORIGINAL 221 BEGIN -->

- \{Host PA \} (0x1a2b3000) ----\> \\text\{IOVA / DMA Address \} (0x80001000)\$\$

<!-- ORIGINAL 221 END -->

<!-- ORIGINAL 222 BEGIN -->

- Trả về địa chỉ IOVA hợp lệ để nạp vào card mạng.

<!-- ORIGINAL 222 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 4.2

```c
dma_addr_t dma = dma_map_page(dev, page, offset, len, DMA_TO_DEVICE);
if (dma_mapping_error(dev, dma)) {
    /* Dừng và xử lý lỗi mapping; không giao địa chỉ lỗi cho NIC. */
}
```

**Ý nghĩa và tham số:** `dev` là NIC cần DMA; `page` là trang RAM; `offset` và `len` xác định vùng; `DMA_TO_DEVICE` nghĩa là dữ liệu đi từ RAM tới NIC. Kết quả dma_addr_t là địa chỉ thiết bị dùng, có thể là IOVA. Nếu mapping SG thì dùng dma_map_sg và số phần tử sau mapping.

**Device và lưu ý:** Không đồng nhất user VA, kernel VA, Host PA và DMA address. DMA API vẫn dùng khi không có IOMMU; một số cấu hình cần bounce qua SWIOTLB. [S7]

<!-- ORIGINAL 224 BEGIN -->

<a id="buoc-4-3"></a>

### Bước 4.3: Điền vòng truyền (TX Ring Buffer)

<!-- ORIGINAL 224 END -->

<!-- ORIGINAL 225 BEGIN -->

**Phần cứng tham gia:** Bộ nhớ RAM (vùng DMA Coherent Memory).

<!-- ORIGINAL 225 END -->

<!-- ORIGINAL 227 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 227 END -->

<!-- ORIGINAL 229 BEGIN -->

- Driver mạng ghi thông tin vào mảng vòng tròn Transmission Ring (TX Ring) nằm trên RAM:

<!-- ORIGINAL 229 END -->

<!-- ORIGINAL 231 BEGIN -->

- Descriptor 0: Trỏ vào địa chỉ Header (TCP/IP/iSCSI Header), cờ FIRST\_FRAG.

<!-- ORIGINAL 231 END -->

<!-- ORIGINAL 233 BEGIN -->

- Descriptor 1: Trỏ vào IOVA 0x80001000 (4096 bytes dữ liệu), cờ LAST\_FRAG.

<!-- ORIGINAL 233 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 4.3

```c
/* Pseudocode — các helper sau không phải API Linux phổ quát. */
fill_tx_descriptor(desc, dma_addr, length, flags);
advance_tx_producer(tx_ring);
```

**Ý nghĩa và tham số:** `desc` là TX descriptor đang chuẩn bị; `dma_addr` là địa chỉ đã map; `length` là số byte; `flags` mô tả segment và offload. tx_ring giữ producer/consumer cùng các metadata để dọn dẹp sau truyền. Mẫu descriptor và helper thật phụ thuộc igb, ixgbe, mlx5 hoặc driver khác.

**Device và lưu ý:** Descriptor phải được công bố với memory ordering phù hợp trước doorbell. Không luôn chỉ có hai descriptor; header, payload fragment, context descriptor và TSO có thể làm số lượng khác đi.

<!-- ORIGINAL 235 BEGIN -->

<a id="buoc-4-4"></a>

### Bước 4.4: Kích hoạt chuông Doorbell qua PCIe MMIO

<!-- ORIGINAL 235 END -->

<!-- ORIGINAL 236 BEGIN -->

**Phần cứng tham gia:** CPU, Bus PCIe (PCIe Root Port → PCIe Switch → Endpoint NIC).

<!-- ORIGINAL 236 END -->

<!-- ORIGINAL 237 BEGIN -->

Cơ chế & Dữ liệu:


<!-- ORIGINAL 237 END -->

<!-- ORIGINAL 238 BEGIN -->

- Để báo cho card mạng biết có gói tin cần gửi, CPU thực thi một lệnh ghi bộ nhớ vào vùng địa chỉ Memory-Mapped I/O (MMIO) của Card NIC:

<!-- ORIGINAL 238 END -->

<!-- ORIGINAL 240 BEGIN -->

- writel(new\_tail\_index,adapter\_bar0+TX\_DOORBELL)

<!-- ORIGINAL 240 END -->

<!-- ORIGINAL 241 BEGIN -->

- Tín hiệu vật lý: CPU gửi một gói tin giao dịch phần cứng PCIe TLP (Transaction Layer Packet) dạng Memory Write (MWr) qua các làn dây (lanes) PCIe cắm trên mainboard tới chip ASIC của card mạng.

<!-- ORIGINAL 241 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 4.4

```c
/* Minh họa accessor MMIO, không phải đoạn driver hoàn chỉnh. */
writel(new_tail, doorbell_reg);
```

**Ý nghĩa và tham số:** `new_tail` là giá trị producer/tail theo định dạng NIC; `doorbell_reg` là con trỏ __iomem tới register đã map. writel ghi giá trị 32 bit theo quy tắc MMIO của nền tảng; không trả kết quả hoàn tất truyền. Từng driver có accessor, ordering và yêu cầu flush riêng.

**Device và lưu ý:** Đây là lệnh CPU báo queue có việc; payload vẫn được NIC đọc qua DMA. Vùng MMIO register và vùng TX descriptor trong RAM là hai vùng khác nhau. [S10]

<!-- ORIGINAL 243 BEGIN -->

<a id="chang-5"></a>

## CHẶNG 5: VẬN CHUYỂN QUA FABRIC & XỬ LÝ TẠI STORAGE TARGET

<!-- ORIGINAL 243 END -->

<!-- ORIGINAL 244 BEGIN -->

(Biến đổi: Đọc DMA từ RAM → Tín hiệu ánh sáng/điện → Chip nhớ Target)

<!-- ORIGINAL 244 END -->

<!-- ORIGINAL 245-260 BEGIN -->

```text
[ Chip điều khiển trên NIC (ASIC) ]
       │ (1. Đọc mô tả từ TX Ring trên RAM)
       ▼
[ NIC DMA Engine: Kích hoạt PCIe Burst Read ]
  - NIC tự rút 4096 bytes từ Host RAM qua Bus PCIe (IOVA -> Host PA)
       │ (2. Ghép Frame Ethernet)
       ▼
[ Bộ chuyển đổi quang/điện (Transceiver SFP+/QSFP) ]
  - Biến đổi nhị phân thành xung ánh sáng (Laser 850nm / 1310nm)
       │
       ▼ (Cáp quang / Switch Leaf-Spine)
[ Cổng mạng trên Storage Target (NetApp) ]
  - Bóc tách Ethernet -> IP -> TCP -> iSCSI PDU -> SCSI CDB
       │
       ▼ (Ghi dữ liệu an toàn)
[ Bộ nhớ đệm Target (NVRAM/Battery-backed Cache) -> Flash NAND ]
```

<!-- ORIGINAL 245-260 END -->

<!-- ORIGINAL 261 BEGIN -->

<a id="buoc-5-1"></a>

### Bước 5.1: Card mạng tự động rút dữ liệu bằng DMA (Scatter-Gather DMA)

<!-- ORIGINAL 261 END -->

<!-- ORIGINAL 262 BEGIN -->

**Phần cứng tham gia:** DMA Engine của NIC, Bus PCIe, IOMMU, Bộ điều khiển bộ nhớ CPU (Memory Controller).

<!-- ORIGINAL 262 END -->

<!-- ORIGINAL 264 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 264 END -->

<!-- ORIGINAL 266 BEGIN -->

- Nhận được tín hiệu Doorbell, chip ASIC của NIC phát các gói tin PCIe Read Request lên bus.

<!-- ORIGINAL 266 END -->

<!-- ORIGINAL 268 BEGIN -->

- Gói tin đi qua IOMMU: IOMMU dịch IOVA 0x80001000 ngược lại thành Host PA 0x1a2b3000.

<!-- ORIGINAL 268 END -->

<!-- ORIGINAL 270 BEGIN -->

- DMA Engine của NIC hút toàn bộ 4096 bytes ký tự A trực tiếp từ thanh RAM vật lý của Host nạp vào bộ đệm truyền dẫn nội bộ (TX FIFO SRAM) nằm trên card mạng.

<!-- ORIGINAL 270 END -->

<!-- ORIGINAL 272 BEGIN -->

- Mức độ chiếm dụng CPU: 0%. CPU hoàn toàn không tham gia vào quá trình di chuyển 4 KiB này.

<!-- ORIGINAL 272 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.1

```c
/* Không có hàm C được CPU gọi cho từng lần NIC thực hiện DMA read.
 * Thao tác phần cứng bắt đầu sau khi descriptor và doorbell hợp lệ. */
```

**Ý nghĩa và tham số:** Đầu vào phần cứng là descriptor: DMA address, số byte và các cờ; đầu ra là payload trong bộ đệm NIC cùng trạng thái TX. NIC tự phát PCIe read, nền tảng dịch địa chỉ DMA khi cần. CPU vẫn làm việc chuẩn bị, xử lý completion và giao thức.

**Device và lưu ý:** Không diễn đạt mức CPU toàn I/O là 0%. Chỉ việc vận chuyển byte bằng DMA không cần CPU chạy vòng memcpy. Dữ liệu có thể được phục vụ qua hệ thống cache coherent, không nhất thiết mỗi byte đều đọc trực tiếp từ chip DRAM.

<!-- ORIGINAL 274 BEGIN -->

<a id="buoc-5-2"></a>

### Bước 5.2: Mã hóa vật lý và Phát tín hiệu quang (Serialization)

<!-- ORIGINAL 274 END -->

<!-- ORIGINAL 275 BEGIN -->

**Phần cứng tham gia:** Tầng MAC/PHY của NIC, Module quang (SFP28 / QSFP28), Sợi cáp quang (Fiber Optic Cable).

<!-- ORIGINAL 275 END -->

<!-- ORIGINAL 277 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 277 END -->

<!-- ORIGINAL 279 BEGIN -->

- Card mạng tính toán mã kiểm tra khung mạng (Ethernet FCS/CRC32).

<!-- ORIGINAL 279 END -->

<!-- ORIGINAL 281 BEGIN -->

- Bộ chuyển đổi tuần tự (Serializer/Deserializer - SerDes) biến đổi các luồng bit nhị phân song song thành luồng tín hiệu nối tiếp tốc độ cao (ví dụ: 10 Gbps hoặc 25 Gbps).

<!-- ORIGINAL 281 END -->

<!-- ORIGINAL 283 BEGIN -->

- Đèn Laser trong module SFP bật/tắt ở tần số hàng gigahertz, biến đổi dòng điện thành các xung ánh sáng (quang tử - photons) truyền dọc theo sợi thủy tinh xuyên qua Switch sang Storage Target.

<!-- ORIGINAL 283 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.2

```c
/* Không có lời gọi hàm host cho từng bit được MAC/PHY phát.
 * MAC/PHY/SerDes xử lý frame đã được NIC chuẩn bị. */
```

**Ý nghĩa và tham số:** Đầu vào là frame/luồng bit và cấu hình link; đầu ra là tín hiệu điện hoặc quang. CRC, encoding, tốc độ symbol và modulation do phần cứng/chế độ link quyết định. Các lời gọi cấu hình link thuộc control path, không phải hàm thực thi cho mỗi I/O.

**Device và lưu ý:** Không phải mọi link đều dùng laser bật/tắt, bước sóng 850/1310 nm hoặc cáp quang; có DAC và nhiều kiểu modulation. Nội dung bản gốc là hình dung một trường hợp cụ thể.

<!-- ORIGINAL 285 BEGIN -->

<a id="buoc-5-3"></a>

### Bước 5.3: Tiếp nhận và Ghi bền vững tại Target

<!-- ORIGINAL 285 END -->

<!-- ORIGINAL 286 BEGIN -->

**Phần cứng tham gia:** Card HBA/NIC Target, CPU Target, NVRAM (Non-Volatile RAM có pin bảo vệ) của Target, Mảng ổ cứng SSD/NAND Flash.

<!-- ORIGINAL 286 END -->

<!-- ORIGINAL 288 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 288 END -->

<!-- ORIGINAL 290 BEGIN -->

- Cổng mạng của Target tiếp nhận ánh sáng, chuyển ngược thành khung frame Ethernet.

<!-- ORIGINAL 290 END -->

<!-- ORIGINAL 292 BEGIN -->

- Kernel/OS của Target (Data ONTAP / LIO Target) bóc tách lớp bọc:

<!-- ORIGINAL 292 END -->

<!-- ORIGINAL 295 BEGIN -->

- Ethernet→IP→TCP→iSCSIPDU→SCSICDB

<!-- ORIGINAL 295 END -->

<!-- ORIGINAL 296 BEGIN -->

- Target phân tích thấy lệnh WRITE\_10 tại LUN chỉ định.

<!-- ORIGINAL 296 END -->

<!-- ORIGINAL 298 BEGIN -->

- Target đổ 4 KiB payload vào NVRAM/DRAM có pin nuôi (Battery-backed Write Cache). Khi dữ liệu đã nằm an toàn trong bộ nhớ bất biến (đảm bảo không mất nếu cúp điện đột ngột), Target coi như lệnh ghi đã hoàn tất (chưa cần đợi nạp điện tích vào từng chip Flash NAND).

<!-- ORIGINAL 298 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.3

```c
/* API target-core minh họa cho Linux LIO; không phải API NetApp. */
void target_execute_cmd(struct se_cmd *cmd);
```

**Ý nghĩa và tham số:** `cmd` là lệnh target đã nhận đủ điều kiện thực thi, giữ device/LUN, CDB và dữ liệu. Target core chuyển xuống backend thích hợp. Tên hàm và backend thực tế phụ thuộc target; không suy ra ONTAP chạy call stack Linux LIO.

**Device và lưu ý:** Ghi thành công phụ thuộc write cache policy, FUA/flush và cam kết bền vững của target. O_DIRECT chỉ liên quan bỏ page cache phía host, không tự bảo đảm dữ liệu đã vào NAND hoặc cache có bảo vệ điện. [S1][S11]

<!-- ORIGINAL 300 BEGIN -->

<a id="chang-6"></a>

## CHẶNG 6: CHIỀU HOÀN TẤT & PHỤC HỒI NGẮNG (COMPLETION PATH)

<!-- ORIGINAL 300 END -->

<!-- ORIGINAL 301 BEGIN -->

(Biến đổi: Phản hồi mạng → Ngắt phần cứng MSI-X → Đánh thức tiến trình Host)

<!-- ORIGINAL 301 END -->

<!-- ORIGINAL 303-321 BEGIN -->

```text
[ Storage Target ]
  - Tạo iSCSI Response PDU (Status: GOOD 0x00, Khớp ITT)
  - Gửi gói tin TCP mang cờ ACK qua mạng
       │
       ▼ (Cáp mạng truyền về Host)
[ Card NIC Host ]
  - DMA ghi bản tin iSCSI Response vào RX Ring trên RAM
  - Kích hoạt ngắt phần cứng MSI-X (Bắn PCIe Memory Write tới LAPIC)
       │
       ▼ (CPU dừng việc khẩn cấp)
[ CPU Core: Chạy Hard-IRQ -> Kích hoạt SoftIRQ (NET_RX_SOFTIRQ) ]
  - TCP bóc gói tin, kiểm tra mã định danh ITT
  - libiscsi -> scsi_mod -> blk-mq đánh dấu request COMPLETE
       │
       ▼ (Giải phóng tài nguyên)
[ Hủy ghim RAM & Trả kết quả cho App ]
  - dma_unmap_page(): Xóa ánh xạ IOMMU
  - unpin_user_page(): Hạ biến đếm refcount trang RAM
  - Trả mã thành công (4096 bytes) qua sysretq / io_getevents
```

<!-- ORIGINAL 303-321 END -->

<!-- ORIGINAL 322 BEGIN -->

<a id="buoc-6-1"></a>

### Bước 6.1: Target gửi phản hồi hoàn tất

<!-- ORIGINAL 322 END -->

<!-- ORIGINAL 323 BEGIN -->

**Phần cứng tham gia:** Card mạng Target → Switch → Card mạng Host.

<!-- ORIGINAL 323 END -->

<!-- ORIGINAL 324 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 324 END -->

<!-- ORIGINAL 325 BEGIN -->

- Target tạo gói tin iSCSI Response PDU:  


<!-- ORIGINAL 325 END -->

<!-- ORIGINAL 326 BEGIN -->

- Opcode = 0x21 (SCSI Response).

<!-- ORIGINAL 326 END -->

<!-- ORIGINAL 328 BEGIN -->

- Status = 0x00 (SAM-4 GOOD - Thành công).

<!-- ORIGINAL 328 END -->

<!-- ORIGINAL 330 BEGIN -->

- Initiator Task Tag (ITT): Trả lại đúng số ITT mà Host đã gửi đi ở Chặng 3.

<!-- ORIGINAL 330 END -->

<!-- ORIGINAL 332 BEGIN -->

- Bọc vào TCP ACK Segment và phát ngược về Host.

<!-- ORIGINAL 332 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 6.1

```c
/* Callback fabric target-core, minh họa cho LIO. */
int queue_status(struct se_cmd *cmd);
```

**Ý nghĩa và tham số:** `cmd` cung cấp status, sense nếu lỗi và thông tin task; fabric tạo response tương ứng gửi initiator. queue_status là tên trường callback, không bảo đảm tên hàm cụ thể giống nhau ở mọi target. ITT giúp ghép response với task; SCSI status GOOD và TCP ACK ở hai lớp giao thức khác nhau.

**Device và lưu ý:** TCP ACK chỉ xác nhận nhận byte theo TCP. Chỉ response/status của giao thức storage mới hoàn tất lệnh SCSI; một ACK thuần không thay thế SCSI Response. [S8][S11]

<!-- ORIGINAL 334 BEGIN -->

<a id="buoc-6-2"></a>

### Bước 6.2: Card NIC Host tiếp nhận & Ghi nhận qua DMA

<!-- ORIGINAL 334 END -->

<!-- ORIGINAL 335 BEGIN -->

**Phần cứng tham gia:** Bộ thu quang PHY của NIC Host, DMA Engine, RAM Host.

<!-- ORIGINAL 335 END -->

<!-- ORIGINAL 337 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 337 END -->

<!-- ORIGINAL 339 BEGIN -->

- Module quang trên Host nhận xung ánh sáng, biến đổi thành tín hiệu điện.

<!-- ORIGINAL 339 END -->

<!-- ORIGINAL 341 BEGIN -->

- NIC DMA Engine ghi thẳng toàn bộ gói tin phản hồi vào một khung trang RAM nằm trong vòng nhận (RX Ring Buffer) của Host.

<!-- ORIGINAL 341 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 6.2

```c
/* Minh họa chuẩn bị buffer RX từ trước. */
dma_addr_t rx_dma = dma_map_page(dev, page, offset, len, DMA_FROM_DEVICE);
```

**Ý nghĩa và tham số:** `dev` là NIC nhận; `page`, `offset`, `len` chỉ vùng nhận; `DMA_FROM_DEVICE` nghĩa NIC ghi vào RAM. Driver chuẩn bị RX descriptor chứa rx_dma trước khi gói đến; NIC sau đó tự DMA. Driver có thể dùng page pool và tái sử dụng mapping thay cho map mỗi packet.

**Device và lưu ý:** RX ring chứa descriptor; packet được DMA vào buffer mà descriptor trỏ tới, không đồng nhất packet payload với chính mảng descriptor. [S7]

<!-- ORIGINAL 343 BEGIN -->

<a id="buoc-6-3"></a>

### Bước 6.3: Kích hoạt Ngắt phần cứng (Hardware Interrupt: MSI-X)

<!-- ORIGINAL 343 END -->

<!-- ORIGINAL 344 BEGIN -->

**Phần cứng tham gia:** PCIe Bus, Bộ điều khiển ngắt lập trình cục bộ của CPU (LAPIC - Local APIC).

<!-- ORIGINAL 344 END -->

<!-- ORIGINAL 346 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 346 END -->

<!-- ORIGINAL 348 BEGIN -->

- Card mạng không kéo chân dây ngắt vật lý cũ (INTx). Nó sử dụng chuẩn MSI-X (Message Signaled Interrupts-Extended).

<!-- ORIGINAL 348 END -->

<!-- ORIGINAL 350 BEGIN -->

- NIC bắn một gói tin PCIe MWr TLP đặc biệt trỏ vào địa chỉ vật lý của bộ nhớ ngắt hệ thống (0xFEE00000 trên kiến trúc x86-64).

<!-- ORIGINAL 350 END -->

<!-- ORIGINAL 352 BEGIN -->

- LAPIC của một CPU Core nhận gói tin này, giải mã vector ngắt, và ép CPU Core đó phải dừng ngay tác vụ tính toán đang chạy để nhảy vào thực thi hàm xử lý ngắt phần cứng (Hard-IRQ).

<!-- ORIGINAL 352 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 6.3

```c
/* API đăng ký handler lúc khởi tạo, không gọi mới cho mỗi packet. */
request_irq(irq, handler, flags, name, dev_id);
```

**Ý nghĩa và tham số:** `irq` là số IRQ Linux; `handler` là hàm driver xử lý; `flags` quy định thuộc tính; `name` là nhãn; `dev_id` là context truyền lại cho handler. Khi MSI-X tới, hạ tầng ngắt gọi handler đã đăng ký. Request_irq trả 0 hoặc lỗi đăng ký.

**Device và lưu ý:** MSI-X có thể được coalesce, mask hoặc remap; không phải cứ một response là một ngắt, cũng không luôn dừng CPU ngay nếu ngắt đang bị chặn. Địa chỉ và vector MSI-X do hệ thống lập trình, không dùng 0xFEE00000 làm địa chỉ cố định cho mọi máy.

<!-- ORIGINAL 354 BEGIN -->

<a id="buoc-6-4"></a>

### Bước 6.4: Xử lý tầng mạng ngầm (SoftIRQ) và Khớp lệnh SCSI

<!-- ORIGINAL 354 END -->

<!-- ORIGINAL 355 BEGIN -->

**Phần cứng tham gia:** CPU Core (chạy ở ngữ cảnh ngắt ksoftirqd).

<!-- ORIGINAL 355 END -->

<!-- ORIGINAL 357 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 357 END -->

<!-- ORIGINAL 359 BEGIN -->

- Hard-IRQ tắt ngắt của card mạng để tránh bão ngắt, lên lịch kích hoạt ngắt mềm NET\_RX\_SOFTIRQ.

<!-- ORIGINAL 359 END -->

<!-- ORIGINAL 361 BEGIN -->

- Ngắt mềm xử lý:

<!-- ORIGINAL 361 END -->

<!-- ORIGINAL 363 BEGIN -->

- Bóc tách TCP Header, cập nhật cửa sổ truyền dẫn.

<!-- ORIGINAL 363 END -->

<!-- ORIGINAL 365 BEGIN -->

- Giao bản tin cho iscsi\_tcp. Driver đọc trường ITT trong PDU, tra cứu bảng băm (Hash Table) để tìm đúng đối tượng struct scsi\_cmnd đang chờ.

<!-- ORIGINAL 365 END -->

<!-- ORIGINAL 367 BEGIN -->

- scsi\_mod xác nhận lệnh thành công, báo lên blk-mq qua hàm blk\_mq\_complete\_request().

<!-- ORIGINAL 367 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 6.4

```c
napi_schedule(&queue->napi);
/* Callback poll minh họa: int poll(struct napi_struct *napi, int budget); */
/* Khi storage stack đã nhận kết quả: */
blk_mq_complete_request(rq);
```

**Ý nghĩa và tham số:** `napi_schedule(napi)` yêu cầu xử lý một NAPI instance; `poll(napi, budget)` xử lý RX theo ngân sách. TCP reassemble byte rồi iscsi-tcp/libiscsi phân tích PDU và ghép ITT. `blk_mq_complete_request(rq)` đánh dấu request cần chạy completion; các tầng trên vẫn phải hoàn tất và trả tài nguyên.

**Device và lưu ý:** SoftIRQ thường chạy trong ngữ cảnh ngắt mềm; có thể chạy qua ksoftirqd hoặc threaded NAPI. Không coi tất cả SoftIRQ đều là ksoftirqd. NAPI có thể poll khi chưa có ngắt. [S12][S4]

<!-- ORIGINAL 369 BEGIN -->

<a id="buoc-6-5"></a>

### Bước 6.5: Dọn dẹp tài nguyên và Đánh thức Ứng dụng

<!-- ORIGINAL 369 END -->

<!-- ORIGINAL 370 BEGIN -->

**Phần cứng tham gia:** IOMMU, MMU CPU, Thanh ghi RAX.

<!-- ORIGINAL 370 END -->

<!-- ORIGINAL 372 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 372 END -->

<!-- ORIGINAL 374 BEGIN -->

- Hủy DMA Mapping: Kernel gọi dma\_unmap\_page(), IOMMU xóa bỏ mục ánh xạ IOVA 0x80001000.

<!-- ORIGINAL 374 END -->

<!-- ORIGINAL 376 BEGIN -->

- Tháo ghim trang RAM: Kernel gọi unpin\_user\_page(), hạ biến đếm của trang RAM vật lý Host PA 0x1a2b3000. Trang RAM này bây giờ có thể được OS tự do quản lý lại bình thường.

<!-- ORIGINAL 376 END -->

<!-- ORIGINAL 378 BEGIN -->

- Kết thúc I/O:

<!-- ORIGINAL 378 END -->

<!-- ORIGINAL 380 BEGIN -->

- Nếu dùng hàm đồng bộ: Luồng ứng dụng đang ngủ trong Kernel được bộ điều phối (Scheduler) đưa về danh sách chạy (TASK\_RUNNING).

<!-- ORIGINAL 380 END -->

<!-- ORIGINAL 382 BEGIN -->

- Nếu dùng io\_submit bất đồng bộ: Kernel ghi một sự kiện thành công vào vòng Completion Ring của AIO (io\_event), hàm io\_getevents() của tiến trình User Space thức dậy.

<!-- ORIGINAL 382 END -->

<!-- ORIGINAL 384 BEGIN -->

- Thanh ghi RAX chứa giá trị 4096. CPU thực thi lệnh sysretq, đưa quyền kiểm soát trở lại cho ứng dụng fio. Chu kỳ 1 lượt I/O ghi trực tiếp chính thức khép lại.

<!-- ORIGINAL 384 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 6.5

```c
dma_unmap_page(dev, dma, len, DMA_TO_DEVICE);
unpin_user_page(page);
/* Completion bio thực tế thường đi qua: */
bio_endio(bio);
/* Native AIO user API minh họa: */
io_getevents(ctx, min_nr, max_nr, events, timeout);
```

**Ý nghĩa và tham số:** `dev`, `dma`, `len`, direction của unmap phải khớp mapping ban đầu; `page` phải được lấy bằng cơ chế pin tương ứng. `bio_endio(bio)` báo bio hoàn tất, có thể kéo theo callback filesystem/direct-I/O. `io_getevents` nhận AIO context, số event tối thiểu/tối đa, mảng kết quả và timeout; mỗi event chứa kết quả riêng, không lấy số byte từ giá trị trả syscall này.

**Device và lưu ý:** NIC dọn DMA mapping ở TX completion khi đã hết quyền đọc dữ liệu, có thể trước SCSI completion và phải xét TCP giữ trang cho retransmit. Pin được tháo khi chủ sở hữu hết dùng. Không gộp mọi lifetime vào cùng một thao tác cuối. write đồng bộ và native AIO có completion khác nhau. [S3][S7][S13]

<!-- ORIGINAL 385 BEGIN -->

<a id="bang-tong-ket"></a>

## BẢNG TỔNG KẾT ĐẶC TẢ SỰ BIẾN ĐỔI QUA 6 CHẶNG

<!-- ORIGINAL 385 END -->

<!-- ORIGINAL 386 BEGIN -->

| Chặng | Dạng tồn tại của Dữ liệu / Tín hiệu | Địa chỉ / Không gian sử dụng | Phần cứng chịu trách nhiệm chính | Có sao chép Payload? |
| --- | --- | --- | --- | --- |
| 1. User → Kernel | Byte stream trong Buffer → struct bio / bio\_vec | User VA → Host PA | CPU Core (ALU, Registers) & MMU | Không (O\_DIRECT ghim trực tiếp RAM) |
| 2. Block & Multipath | struct bio → struct request (Cloned) | Host PA (Gắn trong con trỏ request) | CPU Core (Kernel Scheduler, RAM) | Không (Chỉ truyền con trỏ metadata) |
| 3. SCSI & iSCSI | SCSI CDB (0x2A) → iSCSI Command PDU (48B) | Host PA (Tổ chức thành SG-List) | CPU Core (Tầng driver lưu trữ) | Không (Header tạo mới, Payload dùng SG-List) |
| 4. Network & DMA Prep | Gói tin TCP/IP (skb) → PCIe TX Descriptor | Host PA → IOVA (DMA Address) | IOMMU phần cứng & Bus PCIe MMIO | Không (skb trỏ trực tiếp trang RAM) |
| 5. Fabric Transfer | Khung frame Ethernet → Xung ánh sáng (Photons) | IOVA → Bus Address → Tín hiệu dây | NIC DMA Engine, Bộ phát quang SFP, Switch | Không (Phần cứng tự bốc qua DMA) |
| 6. Completion Path | iSCSI Response → Tín hiệu MSI-X → Trả kết quả | PCIe Interrupt MWr → Register RAX | Card NIC, PCIe Bus, LAPIC, CPU Core | Không (Chỉ truyền tín hiệu trạng thái) |

<!-- ORIGINAL 386 END -->

<!-- ORIGINAL 387 BEGIN -->

Nhận định của bạn hoàn toàn chính xác: các kỹ sư thiết kế Linux Kernel và phần cứng từ nhiều thập kỷ qua đều hướng tới một mục tiêu duy nhất là Zero-Copy (không sao chép payload). Dữ liệu thực chất chỉ nằm cố định tại một vùng RAM vật lý, toàn bộ các tầng phía sau chủ yếu tạo ra các cấu trúc metadata và con trỏ để chỉ đường cho phần cứng tự đến lấy.

<!-- ORIGINAL 387 END -->

<!-- ORIGINAL 389 BEGIN -->

Dưới đây là câu trả lời chi tiết cho cả 3 thắc mắc của bạn:

<!-- ORIGINAL 389 END -->

<!-- ORIGINAL 391 BEGIN -->

<a id="giai-dap-1"></a>

## 1. Nếu lưu luôn trên Local Disk (không qua mạng) thì khác ở đâu?

<!-- ORIGINAL 391 END -->

<!-- ORIGINAL 392 BEGIN -->

Khác biệt lớn nhất là chuỗi mắt xích bị cắt ngắn hơn một nửa. Toàn bộ tầng giao thức mạng, đóng gói và cáp truyền dẫn bị triệt tiêu:

<!-- ORIGINAL 392 END -->

<!-- ORIGINAL 393-396 BEGIN -->

```text
[ GHI QUA MẠNG (iSCSI Baseline) ]
VFS / blk-mq ──> dm-multipath ──> SCSI (CDB) ──> iscsi_tcp (PDU) ──> TCP/IP Stack ──> NIC Driver ──> Cáp mạng ──> NetApp
[ GHI TRỰC TIẾP LOCAL (NVMe SSD cắm trên Host) ]
VFS / blk-mq ──> NVMe Driver (Tạo SQE 64B) ──> Gõ Doorbell thanh ghi PCIe SSD ──> SSD DMA kéo dữ liệu
```

<!-- ORIGINAL 393-396 END -->

<!-- ORIGINAL 397 BEGIN -->

Những thành phần bị loại bỏ khi chạy Local Disk:

<!-- ORIGINAL 397 END -->

<!-- ORIGINAL 398 BEGIN -->

- Không còn dm-multipath: Không cần thuật toán chọn đường (Path Selector), không cần nhân bản request (cloned request).

<!-- ORIGINAL 398 END -->

<!-- ORIGINAL 399 BEGIN -->

- Không còn chuyển đổi SCSI/iSCSI: Nếu dùng SSD NVMe local, Kernel không cần dịch từ bio sang SCSI CDB (WRITE\_10), rồi lại bọc CDB vào iSCSI PDU 48 bytes. Driver NVMe build thẳng lệnh NVMe 64 bytes từ struct request.

<!-- ORIGINAL 399 END -->

<!-- ORIGINAL 400 BEGIN -->

- Không còn Linux TCP/IP Stack: Triệt tiêu hoàn toàn chi phí tạo struct sk\_buff, tính toán TCP Checksum, quản lý cửa sổ trượt (TCP Window), gói tin ACK và giải thuật tránh nghẽn mạng.

<!-- ORIGINAL 400 END -->

<!-- ORIGINAL 401 BEGIN -->

- Không còn Card mạng (NIC) & Cáp truyền dẫn: Thay vì CPU gõ Doorbell lên Card NIC → NIC phát sóng quang qua Switch → Tủ NetApp nhận → NetApp Controller xử lý; thì CPU Host gõ Doorbell trực tiếp vào thanh ghi PCIe của chính chiếc SSD cắm trên bo mạch chủ. Chip điều khiển trên SSD sẽ tự rút dữ liệu từ RAM của Host qua bus PCIe nội bộ.

<!-- ORIGINAL 401 END -->

<!-- ORIGINAL 403 BEGIN -->

<a id="giai-dap-2"></a>

## 2. Dữ liệu chỉ nằm ở buffer đầu tiên, còn lại chỉ là con trỏ chỉ vào?

<!-- ORIGINAL 403 END -->

<!-- ORIGINAL 404 BEGIN -->

Đúng 100% nếu dùng O\_DIRECT, và Đúng 99% nếu dùng Buffered I/O (tính từ sau Page Cache):

<!-- ORIGINAL 404 END -->

<!-- ORIGINAL 406 BEGIN -->

<a id="truong-hop-a"></a>

### Trường hợp A: Dùng O\_DIRECT (Cơ chế trong bài đo của bạn)

<!-- ORIGINAL 406 END -->

<!-- ORIGINAL 407 BEGIN -->

Dữ liệu 4096 bytes nằm chết dí đúng 1 vị trí trên thanh RAM do ứng dụng cấp phát từ đầu chí cuối:

<!-- ORIGINAL 407 END -->

<!-- ORIGINAL 409 BEGIN -->

- Ứng dụng tạo buffer tại địa chỉ ảo User VA.

<!-- ORIGINAL 409 END -->

<!-- ORIGINAL 411 BEGIN -->

- Kernel gọi pin\_user\_pages(), dùng MMU dịch User VA → Host PA và khóa cứng khung trang RAM đó lại.

<!-- ORIGINAL 411 END -->

<!-- ORIGINAL 413 BEGIN -->

- struct bio tạo ra: chỉ chứa con trỏ trỏ tới Host PA đó.

<!-- ORIGINAL 413 END -->

<!-- ORIGINAL 415 BEGIN -->

- struct request tạo ra: chỉ liên kết danh sách các con trỏ bio.

<!-- ORIGINAL 415 END -->

<!-- ORIGINAL 417 BEGIN -->

- SG-List tạo ra: chỉ chứa cặp \[Host PA, 4096\].

<!-- ORIGINAL 417 END -->

<!-- ORIGINAL 419 BEGIN -->

- Socket TCP (sk\_buff) tạo ra: dùng tính năng skb\_frag\_t, trỏ thẳng vào Host PA đó (Zero-copy socket).

<!-- ORIGINAL 419 END -->

<!-- ORIGINAL 421 BEGIN -->

- IOMMU dịch Host PA → IOVA (DMA Address).

<!-- ORIGINAL 421 END -->

<!-- ORIGINAL 423 BEGIN -->

- Controller phần cứng (NIC hoặc SSD): Nhận con trỏ IOVA, tự kích hoạt DMA Engine đọc thẳng 4096 bytes từ đúng ô RAM ban đầu mà ứng dụng đã tạo.

<!-- ORIGINAL 423 END -->

<!-- ORIGINAL 425 BEGIN -->

<a id="truong-hop-b"></a>

### Trường hợp B: Dùng Buffered I/O (Mặc định không cờ)

<!-- ORIGINAL 425 END -->

<!-- ORIGINAL 426 BEGIN -->

- Có 1 lần copy duy nhất: CPU dùng lệnh copy\_from\_user để copy 4096 bytes từ Buffer ứng dụng sang Page Cache của Kernel.

<!-- ORIGINAL 426 END -->

<!-- ORIGINAL 428 BEGIN -->

- Kể từ Page Cache trở đi: Dữ liệu nằm chết dí trong khung trang Page Cache đó. Toàn bộ các bước sau (bio → request → DMA Mapping → Hardware DMA) hoàn toàn chỉ truyền con trỏ trỏ vào khung trang Page Cache, không hề có thêm lần copy payload nào nữa.

<!-- ORIGINAL 428 END -->

<!-- ORIGINAL 430 BEGIN -->

<a id="giai-dap-3"></a>

## 3. Đếm chính xác: Có mấy lần memcpy payload?

<!-- ORIGINAL 430 END -->

<!-- ORIGINAL 431 BEGIN -->

Khi làm báo cáo kỹ thuật, bạn cần phân biệt rạch ròi:

<!-- ORIGINAL 431 END -->

<!-- ORIGINAL 433 BEGIN -->

- Payload memcpy: Sao chép nội dung dữ liệu thật (4096 bytes dữ liệu).

<!-- ORIGINAL 433 END -->

<!-- ORIGINAL 435 BEGIN -->

- Metadata copy: Sao chép các cấu trúc lệnh (mô tả vài chục bytes như CDB, SQE, Headers).

<!-- ORIGINAL 435 END -->

<!-- ORIGINAL 437 BEGIN -->

Dưới đây là số lần Payload memcpy thực tế trên CPU của Host:

<!-- ORIGINAL 437 END -->

<!-- ORIGINAL 439 BEGIN -->

| Kịch bản I/O | Số lần Payload memcpy trên Host | Chi tiết các vị trí copy |
| --- | --- | --- |
| 1. Local Disk + O\_DIRECT | 0 LẦN (Zero-Copy tuyệt đối) | Dữ liệu nằm ở User Buffer → SSD tự dùng DMA kéo thẳng từ RAM qua bus PCIe. |
| 2. Local Disk + Buffered I/O | 1 LẦN | Lần 1: User Buffer → Page Cache (copy\_from\_user). Từ Page Cache xuống SSD là phần cứng tự DMA. |
| 3. iSCSI Network + O\_DIRECT | 0 LẦN (Zero-Copy trên Host) | User Buffer → Kernel ghim RAM → Socket trỏ con trỏ trang → Card NIC tự DMA kéo dữ liệu đẩy ra cáp. |
| 4. iSCSI Network + Buffered I/O | 1 LẦN | Lần 1: User Buffer → Page Cache. Từ Page Cache vào TCP Socket và ra NIC truyền hoàn toàn bằng con trỏ trang (Scatter-Gather DMA). |

<!-- ORIGINAL 439 END -->

<!-- ORIGINAL 440 BEGIN -->

<a id="ngoai-le"></a>

### Ngoại lệ: Khi nào số lần memcpy bị tăng lên?

<!-- ORIGINAL 440 END -->

<!-- ORIGINAL 441 BEGIN -->

Chỉ có 2 trường hợp lỗi cấu hình/phần cứng mới khiến Kernel buộc phải sinh thêm lần copy payload:

<!-- ORIGINAL 441 END -->

<!-- ORIGINAL 443 BEGIN -->

- Bộ đệm bị lệch biên (Unaligned Memory):

<!-- ORIGINAL 443 END -->

<!-- ORIGINAL 444 BEGIN -->

- Khi dùng O\_DIRECT, nếu buffer không chia hết cho 512 bytes hoặc 4096 bytes, hoặc kích thước I/O không chuẩn sector, Kernel hoặc QEMU không thể gán trực tiếp cho phần cứng DMA.

<!-- ORIGINAL 444 END -->

<!-- ORIGINAL 445 BEGIN -->

- Lúc này hệ thống buộc phải tạo ra một vùng nhớ tạm gọi là Bounce Buffer và gọi memcpy toàn bộ payload sang đó trước khi giao cho phần cứng.

<!-- ORIGINAL 445 END -->

<!-- ORIGINAL 447 BEGIN -->

- Tại đầu nhận phía Storage Target (NetApp / LIO):

<!-- ORIGINAL 447 END -->

<!-- ORIGINAL 449 BEGIN -->

- Bảng trên tính trên Compute Host.

<!-- ORIGINAL 449 END -->

<!-- ORIGINAL 451 BEGIN -->

- Khi gói tin sang tới Target, card mạng của Target DMA gói tin vào RAM Target. Nếu phần mềm Target (ví dụ một iSCSI target viết chưa tối ưu) không hỗ trợ Zero-copy từ socket buffer sang block backend, CPU của Target có thể tốn thêm 1 lần memcpy nội bộ trước khi ghi xuống đĩa.

<!-- ORIGINAL 451 END -->

<!-- ORIGINAL 453 BEGIN -->

<a id="ket-luan-goc"></a>

## Kết luận quan trọng cho bài báo cáo của bạn

<!-- ORIGINAL 453 END -->

<!-- ORIGINAL 454 BEGIN -->

Nút thắt cổ chai khiến iSCSI truyền thống chậm hơn NVMe/SPDK và khiến VM-write chậm hơn Host-write KHÔNG PHẢI do CPU tốn công memcpy payload.

<!-- ORIGINAL 454 END -->

<!-- ORIGINAL 456 BEGIN -->

Bản chất payload đều được truyền bằng con trỏ và kéo bằng DMA. Điểm nghẽn thực sự nằm ở:

<!-- ORIGINAL 456 END -->

<!-- ORIGINAL 458 BEGIN -->

- Chi phí xử lý Metadata: Quá nhiều tầng bọc/mở gói (CDB, PDU, TCP, IP).

<!-- ORIGINAL 458 END -->

<!-- ORIGINAL 460 BEGIN -->

- Tranh chấp khóa (Lock Contention): Các CPU Core tranh nhau các hàng đợi của Kernel.

<!-- ORIGINAL 460 END -->

<!-- ORIGINAL 462 BEGIN -->

- Chuyển đổi ngữ cảnh & Ngắt: VM-Exit, Syscall, SoftIRQ, và MSI-X Interrupt làm vỡ vụn bộ nhớ đệm CPU L1/L2/L3 (Cache Pollution).

<!-- ORIGINAL 462 END -->


<a id="device-ao"></a>

## Bổ sung các device logic và device ảo

`/dev/...` là tên node mà tiến trình mở để giao tiếp với driver. Tên node không tự chứng minh dữ liệu nằm trên ổ vật lý cắm tại máy. Cần luôn ghi rõ đang quan sát từ Host hay Guest.

| Device hoặc thành phần | Nhìn từ đâu | Bản chất và vai trò |
| --- | --- | --- |
| `/dev/mapper/mpath0` | Host | Device block logic của dm-multipath, đại diện một LUN qua nhiều path. Tên alias có thể khác. |
| `/dev/dm-0` | Host | Node kernel của một Device Mapper device; có thể là device mà alias mpath0 trỏ đến. Chỉ số không cố định. |
| `/dev/sda`, `/dev/sdb` | Host trong ví dụ gốc | SCSI disk path tới remote LUN; chữ sd không bảo đảm là disk local. Hai path cùng WWID có thể tới cùng dữ liệu. |
| iSCSI session và SCSI host | Host | Thành phần logic nối initiator với target portal và LUN, không phải một SSD vật lý riêng. |
| LUN | Initiator và Target | Đơn vị lưu trữ logic do target xuất; target có thể ánh xạ nó tới pool, volume, file hoặc block backend. |
| `/dev/vda`, `/dev/vdb`, `/dev/vdc` | Guest dùng virtio-blk | Device block ảo; driver guest gửi yêu cầu qua virtqueue tới backend. `/dev/vdb` của Guest không phải `/dev/sdb` của Host. |
| `/dev/sda`, `/dev/sdb` | Guest dùng virtio-scsi | Device SCSI bên trong Guest; vẫn có thể là ổ ảo do hypervisor cung cấp. Tên giống Host không có nghĩa cùng node. |
| `/dev/nvme0n1` | Host hoặc Guest | Namespace NVMe; có thể local PCIe, NVMe-oF hoặc controller NVMe ảo tùy cấu hình. Cần xem transport. |
| `/dev/mapper/vg-lv`, `/dev/loop0`, `/dev/md0` | Host hoặc Guest | Ví dụ device logic khác: LVM, loop ánh xạ file, software RAID. Không mặc nhiên xuất hiện trong luồng gốc. |
| `/dev/vhost-net`, `/dev/vhost-scsi` | Host | Character device cho backend vhost kernel tương ứng; không phải node block chứa payload của volume. |
| Socket vhost-user | Host | Kênh điều khiển Unix socket giữa QEMU và backend user space, không phải node disk Guest. |

**Đối chiếu theo từng chặng:** Chặng 1 mở device logic hoặc file; chặng 2 Device Mapper chọn path; chặng 3 SCSI/iSCSI chuyển lệnh tới LUN; chặng 4 và 5 NIC truyền dữ liệu; chặng 6 completion đi ngược lên cùng yêu cầu logic. Host-write không phải đi qua virtio hay QEMU chỉ vì device đích là dm-multipath.

**Nếu bổ sung VM-write:** Luồng điển hình với virtio-blk là ứng dụng Guest → VFS/direct-I/O Guest → blk-mq Guest → virtio-blk → virtqueue → backend QEMU → file hoặc block device Host → luồng Host ở trên. RAM Guest được backend ánh xạ vào không gian địa chỉ Host; Guest VA, Guest PA, Host VA và Host PA phải được phân biệt. GPA trong descriptor không trực tiếp là DMA address NIC.

Driver virtio-blk có các điểm xử lý như `virtblk_queue_rq` và API virtqueue để thêm descriptor rồi notify; đây là symbol cần kiểm tra theo kernel Guest. Descriptor chứa header thao tác, segment payload và byte status. Backend xử lý queue, đọc/ghi device backing rồi cập nhật used ring để Guest hoàn tất. Tham số virtqueue API gồm queue đích, các scatterlist vào/ra, số list và token nhận lại lúc completion; không đồng nhất token với địa chỉ payload.

**Vhost cần ghi đúng loại:** virtio-blk thông thường có thể dùng QEMU làm backend. Không tự suy ra disk dùng vhost chỉ vì VM có `/dev/vhost-net`, vốn dành cho mạng. vhost-scsi, vhost-user-blk và vhost-user-scsi là các backend khác nhau. Với vhost-user-blk, QEMU cung cấp device virtio cho Guest, trao vùng RAM và thông báo queue cho backend qua giao thức vhost-user; SPDK có thể xử lý I/O storage ở user space. [S14][S15]

Các RPC cấu hình SPDK như `vhost_create_blk_controller` tạo controller gắn một bdev; chúng thuộc bước cấu hình, không phải hàm được gọi cho mỗi write. Đường SPDK có thể đi xuống bdev NVMe-oF, còn đường QEMU dùng raw block device có thể đi xuống dm-multipath. Hai kiến trúc cần trace riêng; tên `/dev/vdb` trong Guest không phân biệt được backend. [S15]

<a id="buffered-io"></a>

## Bổ sung đối chiếu đường buffered I O

Đường buffered minh họa: `write(fd, buf, count)` → VFS → callback filesystem `write_iter` → copy dữ liệu từ user buffer vào page cache → đánh dấu dirty → writeback → tạo bio từ các trang cache → block/device path. Một filesystem có thể dùng helper iomap, filesystem khác dùng helper riêng. `write()` thành công có thể chỉ nghĩa dữ liệu đã vào page cache; `fsync(fd)` yêu cầu đồng bộ theo ngữ nghĩa filesystem và storage stack. [S16]

| Điểm đối chiếu | Buffered I O | Direct I O |
| --- | --- | --- |
| Buffer dùng cho block payload | Thường là trang page cache | Thường là trang buffer ứng dụng được giữ cho I/O |
| Copy user buffer | Thường có copy vào page cache | Thường tránh được lần copy vào page cache |
| Lúc tạo bio | Khi writeback hoặc đồng bộ | Trong đường direct I/O sau kiểm tra/mapping |
| Quan hệ với persistence | Cần xét fsync, flush và cache target | Cũng cần xét sync, FUA, flush và cache target |
| Bảo đảm zero-copy toàn đường | Không có bảo đảm chung | O_DIRECT không phải bảo đảm chung |

`fsync(fd)` nhận descriptor file và trả 0 khi thành công hoặc -1/errno khi lỗi. I/O completion, TCP ACK, SCSI GOOD và persistence là những mốc khác nhau; không dùng chúng thay thế cho nhau. [S1][S8]

<a id="dieu-kien-memcpy"></a>

## Bổ sung điều kiện khi đếm memcpy

Các bảng và kết luận nguyên văn phía trên là mô hình đường tối ưu. Chưa có trace nên chưa xác nhận được số copy trên môi trường thực tế. Hãy phân biệt CPU payload copy, metadata copy và truyền byte bằng DMA. DMA vẫn vận chuyển dữ liệu; zero-copy thường nói đến tránh bản sao payload trong RAM do CPU thực hiện.

Số copy có thể tăng bởi fallback của network path, bounce/SWIOTLB, mã hóa, integrity xử lý, backend file/image, target hoặc cách ánh xạ bộ nhớ VM. Vì vậy danh sách hai ngoại lệ trong bản gốc không phải danh sách đầy đủ. Alignment không hợp lệ cũng không luôn tạo bounce buffer: tùy filesystem/kernel có thể lỗi hoặc fallback buffered. [S1][S7][S9]

Local NVMe thường bỏ SCSI/iSCSI/TCP/NIC của đường remote, nhưng vẫn có thể dùng Device Mapper hoặc lớp logic khác nếu cấu hình. Không kết luận local disk luôn không có dm-multipath hay không có mapping trung gian.

Khi đo, ghi lại kernel, filesystem, block size, target cache policy, Device Mapper mode, NIC/offload, QEMU/backend và cấu hình Guest. Theo dõi submit/complete ở block và TX/RX ở mạng để xác định lifetime. Dùng trace/counter phù hợp để xác nhận copy; chỉ nhìn một symbol memcpy không đủ phát hiện copy được inline hoặc qua helper khác. Cũng không kết luận bottleneck chắc chắn là metadata/ngắt hay chắc chắn không do copy khi chưa đo.

<a id="nguon-tham-khao"></a>

## Nguồn tham khảo cho nội dung bổ sung


- **[S1]** [Linux man pages open O_DIRECT](https://man7.org/linux/man-pages/man2/open.2.html).
- **[S2]** [Linux man pages posix_memalign](https://man7.org/linux/man-pages/man3/posix_memalign.3.html).
- **[S3]** [Linux pin_user_pages và các API liên quan](https://kernel.org/doc/html/latest/core-api/pin_user_pages.html).
- **[S4]** [Linux 6.8 Multi Queue Block I O](https://www.kernel.org/doc/html/v6.8/block/blk-mq.html).
- **[S5]** [Red Hat Device Mapper multipath và device logic](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/dm_multipath/mpath_devices).
- **[S6]** [Linux SCSI mid và low level API](https://www.kernel.org/doc/html/v6.8/scsi/scsi_mid_low_api.html).
- **[S7]** [Linux 6.8 Dynamic DMA mapping Guide](https://www.kernel.org/doc/html/v6.8/core-api/dma-api-howto.html).
- **[S8]** [RFC 7143 iSCSI Protocol](https://www.rfc-editor.org/rfc/rfc7143.html).
- **[S9]** [Linux networking API và MSG_SPLICE_PAGES](https://cdn.kernel.org/doc/html/latest/networking/kapi.html).
- **[S10]** [Linux Bus Independent Device Accesses](https://www.kernel.org/doc/html/latest/driver-api/device-io.html).
- **[S11]** [Linux 6.8 SCSI target API](https://www.kernel.org/doc/html/v6.8/driver-api/target.html).
- **[S12]** [Linux 6.8 NAPI](https://www.kernel.org/doc/html/v6.8/networking/napi.html).
- **[S13]** [Linux man pages io_getevents](https://man7.org/linux/man-pages/man2/io_getevents.2.html).
- **[S14]** [QEMU Vhost user Protocol](https://www.qemu.org/docs/master/interop/vhost-user.html).
- **[S15]** [SPDK vhost Target](https://spdk.io/doc/vhost.html).
- **[S16]** [Linux iomap file operations](https://www.kernel.org/doc/html/latest/filesystems/iomap/operations.html).
