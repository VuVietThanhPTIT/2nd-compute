# Luồng VM write qua virtio QEMU và iSCSI multipath



## Mục lục

- [Mở đầu ](#mo-dau)
- [CHẶNG 1: GUEST OS — TỪ APP ĐẾN HÀNG ĐỢI VIRTQUEUE](#chang-1)
  - [Bước 1.1: Ứng dụng Guest phát lệnh ghi trực tiếp](#buoc-1-1)
  - [Bước 1.2: Xử lý tại Guest Kernel & Ghim trang RAM](#buoc-1-2)
  - [Bước 1.3: Driver virtio-blk điền bảng Virtqueue](#buoc-1-3)
- [CHẶNG 2: RANH GIỚI ẢO HÓA — CÚ NHẢY VM-EXIT VÀ BÁO HIỆU QEMU](#chang-2)
  - [tBước 2.1: Gõ chuông ảo (Virtual Doorbell MMIO)](#buoc-2-1)
  - [Bước 2.2: Bẫy phần cứng CPU: Sự kiện VM-Exit](#buoc-2-2)
  - [Bước 2.3: KVM xử lý ngắt và bắn tín hiệu ioeventfd](#buoc-2-3)
  - [Bước 2.4: Đánh thức QEMU IOThread (Context Switch)](#buoc-2-4)
- [CHẶNG 3: QEMU IOTHREAD — DỊCH ĐỊA CHỈ VÀ PHÁT LỆNH HOST SYSCALL](#chang-3)
  - [Bước 3.1: QEMU đọc cấu trúc Virtqueue](#buoc-3-1)
  - [Bước 3.2: Dịch địa chỉ bộ nhớ hai cấp (GPA → HVA)](#buoc-3-2)
  - [Bước 3.3: Phát lời gọi hệ thống Linux Native AIO (io\_submit)](#buoc-3-3)
- [CHẶNG 4: HOST STORAGE STACK & MẠNG iSCSI](#chang-4)
- [CHẶNG 5: CHIỀU HOÀN TẤT VÀ VÒNG LẶP TIÊM NGẮT ẢO (COMPLETION PATH)](#chang-5)
  - [Bước 5.1: Mạng Host nhận kết quả và ngắt phần cứng](#buoc-5-1)
  - [Bước 5.2: QEMU IOThread nhận hoàn tất & Cập nhật Used Ring](#buoc-5-2)
  - [Bước 5.3: Báo hiệu qua irqfd và KVM tiêm ngắt ảo](#buoc-5-3)
  - [Bước 5.4: Guest kết thúc chu trình](#buoc-5-4)
- [CHẶNG 6: TỔNG KẾT VI PHẪU RANH GIỚI ẢO HÓA (OVERHEAD BREAKDOWN)](#chang-6)
  - [1. Bản đồ biến đổi địa chỉ 5 cấp (The 5-Level Address Chain)](#tong-ket-1)
  - [2. Định lượng các cú nhảy ngữ cảnh & Bẫy CPU (Context Switches & Traps)](#tong-ket-2)
  - [3. Phân định rạch ròi số lần memcpy trong Luồng VM-write](#tong-ket-3)
- [KẾT LUẬN:](#ket-luan-goc)
- [Bổ sung device và các không gian địa chỉ](#device-ao)
- [Bổ sung điều kiện của các kết luận overhead](#dieu-kien-overhead)
- [Nguồn tham khảo cho nội dung bổ sung](#nguon-tham-khao)

<a id="mo-dau"></a>

![alt text](image.png)
## Mở đầu nguyên văn

<!-- ORIGINAL 0 BEGIN -->

Để làm rõ ranh giới ảo hóa làm phình to overhead và gây trễ I/O ở những bước nào, ta mổ xẻ luồng VM-write theo mô hình chuẩn:

<!-- ORIGINAL 0 END -->

<!-- ORIGINAL 2 BEGIN -->

- Ứng dụng trong Guest VM chạy lệnh write() với cờ O\_DIRECT.

<!-- ORIGINAL 2 END -->

<!-- ORIGINAL 3 BEGIN -->

- Cấu hình đĩa ảo tại QEMU/libvirt: cache=none (Host mở thiết bị bằng O\_DIRECT, bỏ qua Page Cache của Host) và io=native (QEMU dùng Linux Native AIO io\_submit thay vì thread pool).

<!-- ORIGINAL 3 END -->

<!-- ORIGINAL 4 BEGIN -->

- Backend trên Compute Host: /dev/mapper/mpath0 nối tới iSCSI Target.

<!-- ORIGINAL 4 END -->

<!-- ORIGINAL 5 BEGIN -->

Toàn bộ quá trình được chia thành 5 chặng thực thi và 1 chặng tổng kết vi phẫu.

<!-- ORIGINAL 5 END -->

<!-- ORIGINAL 7 BEGIN -->

<a id="chang-1"></a>

## CHẶNG 1: GUEST OS — TỪ APP ĐẾN HÀNG ĐỢI VIRTQUEUE

<!-- ORIGINAL 7 END -->

<!-- ORIGINAL 8 BEGIN -->

(Biến đổi: Vùng đệm Guest → Syscall nội bộ VM → Bảng giao việc Virtqueue trên RAM)

<!-- ORIGINAL 8 END -->

<!-- ORIGINAL 10-24 BEGIN -->

```text
[ Ứng dụng Guest (fio) ]
  - Buffer 4096B (Guest VA)
  - Gọi syscall write(fd, buf, 4096) với O_DIRECT
       │
       ▼ (Chuyển quyền nội bộ Guest: Ring 3 -> Ring 0)
[ Guest Kernel: VFS & blk-mq ]
  - Bỏ qua Page Cache Guest (O_DIRECT)
  - MMU Guest dịch: Guest VA -> Guest PA (GPA)
  - Khóa trang RAM Guest (pin_user_pages)
  - Cấp phát struct bio -> struct request
       │
       ▼ (Driver virtio-blk)
[ Virtqueue (Split Ring trên RAM Guest) ]
  - Descriptor Table: Desc 0 (Header) -> Desc 1 (GPA buffer) -> Desc 2 (Status)
  - Available Ring: Nạp chỉ số Desc 0, tăng avail->idx
```

<!-- ORIGINAL 10-24 END -->

<!-- ORIGINAL 25 BEGIN -->

<a id="buoc-1-1"></a>

### Bước 1.1: Ứng dụng Guest phát lệnh ghi trực tiếp

<!-- ORIGINAL 25 END -->

<!-- ORIGINAL 26 BEGIN -->

**Phần cứng tham gia:** vCPU của Guest (chạy ở chế độ phần cứng VMX Non-Root Mode), RAM cấp phát cho Guest.

<!-- ORIGINAL 26 END -->

<!-- ORIGINAL 28 BEGIN -->

Cơ chế & Dữ liệu:


<!-- ORIGINAL 28 END -->

<!-- ORIGINAL 29 BEGIN -->

- Ứng dụng cấp phát bộ đệm 4096 bytes căn chỉnh 4 KiB tại địa chỉ ảo Guest Virtual Address (GVA) (ví dụ: 0x00401000).

<!-- ORIGINAL 29 END -->

<!-- ORIGINAL 30 BEGIN -->

- Ứng dụng nạp thanh ghi và phát chỉ lệnh syscall.

<!-- ORIGINAL 30 END -->

<!-- ORIGINAL 31 BEGIN -->

- Chuyển quyền nội bộ: vCPU chuyển đặc quyền từ Guest Ring 3 sang Guest Ring 0. Lưu ý: Đây là chuyển quyền ảo hóa bên trong chế độ Non-Root, chưa gây ra VM-Exit.

<!-- ORIGINAL 31 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.1

```c
int fd = open("/dev/vdb", O_WRONLY | O_DIRECT);
void *buf = NULL;
int rc = posix_memalign(&buf, 4096, 4096);
/* Chỉ tiếp tục khi fd >= 0 và rc == 0. */
memset(buf, 'A', 4096);
ssize_t n = pwrite(fd, buf, 4096, 0);
```

**Ý nghĩa và tham số:** `open(path, flags)` trả descriptor của device Guest; `posix_memalign(&buf, alignment, size)` cấp buffer căn chỉnh, trả mã lỗi trực tiếp. `memset(buf, value, count)` tạo payload. `pwrite(fd, buf, count, offset)` ghi tại offset mà không đổi file offset dùng chung, trả số byte hoặc lỗi. Đây là ví dụ ghi vào device, không nên chạy trên volume đang dùng. Kiểm tra short write và chỉ giải phóng buffer sau completion.

**Device và lưu ý:** `/dev/vdb` là device virtio-blk trong Guest, không phải path iSCSI trên Host. Yêu cầu alignment phụ thuộc filesystem/device; 4096 là lựa chọn minh họa. O_DIRECT Guest và cache=none Host là hai cấu hình riêng, cần kiểm tra cả hai. [V1][V2]

<!-- ORIGINAL 32 BEGIN -->

<a id="buoc-1-2"></a>

### Bước 1.2: Xử lý tại Guest Kernel & Ghim trang RAM

<!-- ORIGINAL 32 END -->

<!-- ORIGINAL 33 BEGIN -->

**Phần cứng tham gia:** Bảng phân trang của Guest (Guest Page Table - GPT).

<!-- ORIGINAL 33 END -->

<!-- ORIGINAL 34 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 34 END -->

<!-- ORIGINAL 36 BEGIN -->

- Do có cờ O\_DIRECT, Guest Kernel bỏ qua Page Cache của Guest.

<!-- ORIGINAL 36 END -->

<!-- ORIGINAL 38 BEGIN -->

- Guest MMU tra cứu bảng trang GPT để dịch:

<!-- ORIGINAL 38 END -->

<!-- ORIGINAL 40 BEGIN -->

- \{Guest VA \} (0x00401000) ----\> \\text\{Guest PA (GPA) \} (0x00105000)\$\$

<!-- ORIGINAL 40 END -->

<!-- ORIGINAL 41 BEGIN -->

- Guest Kernel ghim khung trang GPA này lại, đóng gói vào struct bio → chuyển thành struct request trong tầng blk-mq của Guest.

<!-- ORIGINAL 41 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.2

```c
/* API pin minh họa; direct I/O có thể dùng helper iov_iter/bio. */
long nr = pin_user_pages_fast(gva, nr_pages, gup_flags, pages);
/* Sau mapping file/device và tạo bio: */
submit_bio(bio);
```

**Ý nghĩa và tham số:** `gva` là địa chỉ user space của Guest; `nr_pages` là số trang; `gup_flags` điều khiển cách lấy trang; `pages` nhận các `struct page *` thuộc Guest. `nr` báo số trang ghim được hoặc lỗi. `submit_bio(bio)` giao I/O cho block layer Guest, completion được trả bất đồng bộ qua callback. Ghi ra device đọc nội dung buffer, không mặc nhiên đặt FOLL_WRITE.

**Device và lưu ý:** Guest pin giữ trang theo quản lý bộ nhớ Guest; không đồng nhất với pin Host để NIC DMA. struct page Guest không phải struct page Host. Trên CPU ảo hóa, truy cập bộ nhớ Guest còn được phần cứng dịch GPA sang HPA qua EPT/NPT; không có một chip MMU Guest tách riêng. [V3]

<!-- ORIGINAL 43 BEGIN -->

<a id="buoc-1-3"></a>

### Bước 1.3: Driver virtio-blk điền bảng Virtqueue

<!-- ORIGINAL 43 END -->

<!-- ORIGINAL 44 BEGIN -->

**Phần cứng tham gia:** Vùng nhớ RAM Guest chia sẻ với Host.

<!-- ORIGINAL 44 END -->

<!-- ORIGINAL 46 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 46 END -->

<!-- ORIGINAL 48 BEGIN -->

- Driver virtio-blk trong Guest lấy request ra và tạo chuỗi 3 phần tử trong Descriptor Table:

<!-- ORIGINAL 48 END -->

<!-- ORIGINAL 50 BEGIN -->

- Desc 0: Chứa struct virtio\_blk\_outhdr (loại lệnh VIRTIO\_BLK\_T\_OUT = ghi, sector bắt đầu trên ổ /dev/vdb). Cờ VRING\_DESC\_F\_NEXT.

<!-- ORIGINAL 50 END -->

<!-- ORIGINAL 52 BEGIN -->

- Desc 1: Chứa địa chỉ dữ liệu payload. Địa chỉ ghi vào đây là GPA 0x00105000, độ dài 4096 bytes. Cờ VRING\_DESC\_F\_NEXT.

<!-- ORIGINAL 52 END -->

<!-- ORIGINAL 54 BEGIN -->

- Desc 2: Chứa địa chỉ 1 byte trạng thái (Status byte) để nhận kết quả. Cờ VRING\_DESC\_F\_WRITE.

<!-- ORIGINAL 54 END -->

<!-- ORIGINAL 56 BEGIN -->

- Cập nhật Available Ring: Driver ghi chỉ số 0 vào mảng avail-\>ring\[idx\] và tăng con trỏ avail-\>idx.

<!-- ORIGINAL 56 END -->

<!-- ORIGINAL 58 BEGIN -->

- Bản chất sao chép: Payload 4 KiB vẫn nằm nguyên tại ô RAM 0x00105000.

<!-- ORIGINAL 58 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 1.3

```c
/* Callback virtio-blk thường gặp, cần đối chiếu kernel Guest. */
blk_status_t virtblk_queue_rq(struct blk_mq_hw_ctx *hctx,
                             const struct blk_mq_queue_data *bd);
/* API đưa scatterlists vào virtqueue minh họa: */
virtqueue_add_sgs(vq, sgs, out_sgs, in_sgs, token, gfp);
```

**Ý nghĩa và tham số:** `hctx` là dispatch context; `bd->rq` là request Guest. `vq` là virtqueue; `sgs` chứa các scatterlist; `out_sgs` là số list device đọc, `in_sgs` là số list device ghi; `token` nhận lại khi thu hồi completion; `gfp` là cờ cấp phát. Request ghi thường có header và payload ở phía device đọc, status ở phía device ghi.

**Device và lưu ý:** Mô hình 3 descriptor chỉ là trường hợp đơn giản. SG, indirect descriptor, packed ring và virtual IOMMU có thể thay đổi layout/địa chỉ. VRING_DESC_F_WRITE nghĩa device được ghi buffer, không có nghĩa thao tác storage là write. Với split ring cần memory ordering trước khi công bố avail idx. [V4]

<!-- ORIGINAL 60 BEGIN -->

<a id="chang-2"></a>

## CHẶNG 2: RANH GIỚI ẢO HÓA — CÚ NHẢY VM-EXIT VÀ BÁO HIỆU QEMU

<!-- ORIGINAL 60 END -->

<!-- ORIGINAL 61 BEGIN -->

(Biến đổi: Ghi thanh ghi ảo → Bẫy phần cứng CPU → KVM đánh thức QEMU IOThread)

<!-- ORIGINAL 61 END -->

<!-- ORIGINAL 62-79 BEGIN -->

```text
[ Driver virtio-blk (Guest) ]
  - Ghi chỉ mục hàng đợi vào thanh ghi MMIO Doorbell
       │
       ▼ (ĐIỂM NGHẼN 1: BẪY PHẦN CỨNG)
[ CPU Phần Cứng: KÍCH HOẠT SỰ KIỆN VM-EXIT ]
  - Chuyển chế độ: VMX Non-Root (Guest) -> VMX Root (Host Kernel)
  - Lưu trạng thái vCPU vào VMCS (Virtual Machine Control Structure)
       │
       ▼ (Host Kernel: KVM Module)
[ Module KVM (kvm.ko) ]
  - Đọc Exit Reason: EPT_MISCONFIG / MMIO Write
  - Tra cứu địa chỉ MMIO -> Khớp với thanh ghi Queue Notify
  - KVM ghi 8-byte vào ioeventfd (Bỏ qua QEMU Main Loop)
       │
       ▼ (ĐIỂM NGHẼN 2: CONTEXT SWITCH LUỒNG)
[ QEMU IOThread (AioContext) ]
  - Luồng đang ngủ tại epoll_wait(ioeventfd) bị đánh thức
  - Chuyển ngữ cảnh sang QEMU IOThread trên CPU Hos
```

<!-- ORIGINAL 62-79 END -->

<!-- ORIGINAL 80 BEGIN -->

<a id="buoc-2-1"></a>

### tBước 2.1: Gõ chuông ảo (Virtual Doorbell MMIO)

<!-- ORIGINAL 80 END -->

<!-- ORIGINAL 81 BEGIN -->

**Phần cứng tham gia:** Bus PCI ảo của VM, thanh ghi MMIO Queue Notify.

<!-- ORIGINAL 81 END -->

<!-- ORIGINAL 82 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 82 END -->

<!-- ORIGINAL 83 BEGIN -->

- Guest điền xong Virtqueue nhưng Host chưa hề hay biết. Để báo hiệu, driver virtio-blk thực hiện một lệnh ghi bộ nhớ vào thanh ghi MMIO ảo:

<!-- ORIGINAL 83 END -->

<!-- ORIGINAL 84 BEGIN -->

- writel(queue\_index,mmio\_doorbell\_address)

<!-- ORIGINAL 84 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.1

```c
virtqueue_kick(vq);
/* Đường notify transport có thể dẫn tới MMIO hoặc port I/O.
 * Ví dụ MMIO accessor: writel(queue_index, notify_reg); */
```

**Ý nghĩa và tham số:** `vq` là queue cần notify. Driver có thể kiểm tra suppression bằng các helper kick_prepare/notify. `queue_index` xác định queue; `notify_reg` là địa chỉ __iomem của register ảo theo transport. Giá trị ghi không phải payload 4096 byte.

**Device và lưu ý:** Notify không nhất thiết xuất hiện cho mỗi request: có batching và event suppression. virtio-pci legacy/modern, virtio-mmio và negotiated features có cách notify khác nhau. writel trong bản gốc là một ví dụ, không là accessor bắt buộc của mọi VM. [V4]

<!-- ORIGINAL 85 BEGIN -->

<a id="buoc-2-2"></a>

### Bước 2.2: Bẫy phần cứng CPU: Sự kiện VM-Exit

<!-- ORIGINAL 85 END -->

<!-- ORIGINAL 86 BEGIN -->

**Phần cứng tham gia:** Vi xử lý CPU x86-64 (Intel VT-x / AMD-V), cấu trúc bộ nhớ phần cứng VMCS (Virtual Machine Control Structure).

<!-- ORIGINAL 86 END -->

<!-- ORIGINAL 88 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 88 END -->

<!-- ORIGINAL 90 BEGIN -->

- Địa chỉ MMIO này không có thật trên bảng phân trang mở rộng (EPT - Extended Page Tables). Khi vCPU cố ghi vào đây, CPU phần cứng lập tức kích hoạt bẫy VM-Exit.

<!-- ORIGINAL 90 END -->

<!-- ORIGINAL 91 BEGIN -->

- Tác động vật lý:

<!-- ORIGINAL 91 END -->

<!-- ORIGINAL 93 BEGIN -->

- CPU phần cứng dừng toàn bộ luồng thực thi của máy ảo.

<!-- ORIGINAL 93 END -->

<!-- ORIGINAL 94 BEGIN -->

- CPU tự động lưu toàn bộ các thanh ghi hiện tại của vCPU vào vùng RAM cấu trúc VMCS.

<!-- ORIGINAL 94 END -->

<!-- ORIGINAL 95 BEGIN -->

- CPU chuyển trạng thái thực thi từ VMX Non-Root sang VMX Root (Host Kernel Space Ring 0).

<!-- ORIGINAL 95 END -->

<!-- ORIGINAL 97 BEGIN -->

- Chi phí trễ: Đây là cú nhảy phần cứng đắt đỏ, tiêu tốn trung bình từ 1.5-2.5μ s cho mỗi lần thoát/nhập (Exit/Entry).

<!-- ORIGINAL 97 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.2

```c
/* API user space đưa vCPU vào chạy, thuộc vCPU thread. */
ioctl(vcpu_fd, KVM_RUN, 0);
/* VM-Exit do CPU/KVM xử lý, không phải hàm C Guest gọi. */
```

**Ý nghĩa và tham số:** `vcpu_fd` là descriptor KVM của vCPU. KVM_RUN cho phép chạy Guest; không phải mỗi exit phần cứng đều làm ioctl này trả về QEMU user space. CPU/KVM giữ trạng thái cần thiết để xử lý exit và resume. Nếu phải emulation ở user space, kvm_run cung cấp exit reason và dữ liệu liên quan.

**Device và lưu ý:** Exit reason có thể khác EPT_MISCONFIG. Không nói phần cứng tự lưu toàn bộ general-purpose registers vào VMCS: một phần trạng thái do mã KVM lưu. VM-Exit dừng vCPU liên quan, không mặc nhiên dừng mọi vCPU của VM. VMCS/VMX là thuật ngữ Intel; AMD dùng VMCB/SVM. [V5]

<!-- ORIGINAL 99 BEGIN -->

<a id="buoc-2-3"></a>

### Bước 2.3: KVM xử lý ngắt và bắn tín hiệu ioeventfd

<!-- ORIGINAL 99 END -->

<!-- ORIGINAL 100 BEGIN -->

**Phần cứng tham gia:** CPU Host (chạy mã kvm-intel.ko / kvm.ko).

<!-- ORIGINAL 100 END -->

<!-- ORIGINAL 102 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 102 END -->

<!-- ORIGINAL 104 BEGIN -->

- KVM kiểm tra trường VM\_EXIT\_REASON, xác định đây là một thao tác ghi MMIO vào virtio-blk.

<!-- ORIGINAL 104 END -->

<!-- ORIGINAL 106 BEGIN -->

- KVM không xử lý I/O mà gọi hàm eventfd\_signal(), ghi một con số 64-bit vào file descriptor nội bộ ioeventfd đã được đăng ký trước đó.

<!-- ORIGINAL 106 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.3

```c
/* Đăng ký từ trước ở control path, ví dụ giản lược. */
ioctl(vm_fd, KVM_IOEVENTFD, &registration);
/* Kernel datapath: eventfd_signal(ctx) hoặc phiên bản có đối số đếm,
 * tùy kernel. Đây không phải write(fd, buf, 8) trong user space. */
```

**Ý nghĩa và tham số:** `vm_fd` là descriptor VM; `registration` là kvm_ioeventfd chứa fd, addr, len, datamatch và flags. `ctx` là eventfd_ctx kernel được giữ từ đăng ký. Khi MMIO/PIO khớp, KVM báo sự kiện cho backend mà có thể không cần thoát tiếp ra vòng emulation QEMU.

**Device và lưu ý:** ioeventfd là cơ chế đếm/báo sự kiện; không vận chuyển 4 KiB payload. Cụm ghi 8-byte trong bản gốc là cách hình dung eventfd user API, không phải KVM ghi một gói 8 byte qua file cho từng I/O. [V5]

<!-- ORIGINAL 108 BEGIN -->

<a id="buoc-2-4"></a>

### Bước 2.4: Đánh thức QEMU IOThread (Context Switch)

<!-- ORIGINAL 108 END -->

<!-- ORIGINAL 109 BEGIN -->

**Phần cứng tham gia:** Bộ điều phối tiến trình Linux (Linux Scheduler).

<!-- ORIGINAL 109 END -->

<!-- ORIGINAL 111 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 111 END -->

<!-- ORIGINAL 113 BEGIN -->

- Do VM được cấu hình object iothread, tiến trình QEMU sở hữu một luồng độc lập (IOThread) chạy vòng lặp sự kiện AioContext (đang treo chờ ở hàm epoll\_wait).

<!-- ORIGINAL 113 END -->

<!-- ORIGINAL 115 BEGIN -->

- Tín hiệu từ ioeventfd làm epoll\_wait trả về.

<!-- ORIGINAL 115 END -->

<!-- ORIGINAL 117 BEGIN -->

- Chi phí trễ: Linux Scheduler phải can thiệp để Context Switch đưa luồng QEMU IOThread lên một CPU Core của Host để chạy. Bộ nhớ đệm CPU L1/L2 của Core đó bị xáo trộn để nạp ngữ cảnh của tiến trình QEMU.

<!-- ORIGINAL 117 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 2.4

```c
/* Ví dụ syscall chờ event; QEMU dùng AioContext và cơ chế polling riêng. */
int nready = epoll_wait(epfd, events, maxevents, timeout);
```

**Ý nghĩa và tham số:** `epfd` là epoll instance; `events` nhận danh sách event; `maxevents` là sức chứa; `timeout` là thời gian chờ. Giá trị trả là số event hoặc lỗi. Event có thể khiến IOThread được scheduler chạy rồi callback xử lý virtqueue được thực hiện.

**Device và lưu ý:** epoll_wait không nhận trực tiếp ioeventfd như một tham số duy nhất. Tạo object iothread chưa đủ: device/queue phải được gắn với IOThread. Luồng có thể đã chạy, dùng polling hoặc gom nhiều request, nên không có một context switch bắt buộc cho mỗi request. [V6]

<!-- ORIGINAL 119 BEGIN -->

<a id="chang-3"></a>

## CHẶNG 3: QEMU IOTHREAD — DỊCH ĐỊA CHỈ VÀ PHÁT LỆNH HOST SYSCALL

<!-- ORIGINAL 119 END -->

<!-- ORIGINAL 120 BEGIN -->

(Biến đổi: GPA của Guest → HVA của QEMU → Linux Native AIO Syscall)

<!-- ORIGINAL 120 END -->

<!-- ORIGINAL 121-140 BEGIN -->

```text
[ QEMU IOThread ]
  - Đọc Available Ring từ RAM chia sẻ
  - Lấy Desc 0, Desc 1, Desc 2
       │
       ▼ (ĐIỂM NGHẼN 3: DỊCH ĐỊA CHỈ HAI CẤP)
[ Ánh xạ Bộ nhớ RAM Máy ảo ]
  - Đọc Desc 1: Thấy dữ liệu tại GPA 0x00105000
  - QEMU tra bảng MemoryRegionSection:
    GPA (0x00105000) ──> Host Virtual Address (HVA: 0x7f11a000)
       │
       ▼ (Tạo Linux Native AIO Request)
[ Cấu trúc struct iocb ]
  - aio_fildes = fd của /dev/mapper/mpath0
  - aio_buf = 0x7f11a000 (HVA)
  - aio_nbytes = 4096
  - aio_lio_opcode = IOCB_CMD_PWRITEV
       │
       ▼ (Chỉ lệnh SYSCALL trên Host)
[ Host Syscall: io_submit() ]
  - Đổi quyền Host: Ring 3 (QEMU) -> Ring 0 (Host Kernel)
```

<!-- ORIGINAL 121-140 END -->

<!-- ORIGINAL 142 BEGIN -->

<a id="buoc-3-1"></a>

### Bước 3.1: QEMU đọc cấu trúc Virtqueue

<!-- ORIGINAL 142 END -->

<!-- ORIGINAL 143 BEGIN -->

**Phần cứng tham gia:** Bộ nhớ RAM Host, CPU L1/L2 Cache.

<!-- ORIGINAL 143 END -->

<!-- ORIGINAL 145 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 145 END -->

<!-- ORIGINAL 146 BEGIN -->

- IOThread của QEMU truy cập vào mảng Available Ring của Virtqueue (vốn nằm trên RAM vật lý của Host mà QEMU đã mmap từ trước).

<!-- ORIGINAL 146 END -->

<!-- ORIGINAL 147 BEGIN -->

- QEMU bốc ra Descriptor chuỗi Desc 0 → Desc 1 → Desc 2.

<!-- ORIGINAL 147 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.1

```c
/* API QEMU nội bộ minh họa; chữ ký phải kiểm tra đúng phiên bản. */
VirtQueueElement *elem = virtqueue_pop(vq, elem_size);
```

**Ý nghĩa và tham số:** `vq` là queue QEMU đang phục vụ; `elem_size` là dung lượng đối tượng phần tử cần cấp. Kết quả là element mô tả các buffer out/in hoặc NULL khi không có phần tử phù hợp. Backend kiểm tra chain và header trước khi chuyển I/O xuống block layer QEMU.

**Device và lưu ý:** Buffer out do backend đọc chứa header/payload, buffer in do backend ghi chứa status. Backend phải kiểm tra quyền, kích thước và địa chỉ descriptor. Không coi descriptor luôn bắt đầu ở chỉ số 0 hay chain luôn có đúng 3 phần tử. [V4]

<!-- ORIGINAL 148 BEGIN -->

<a id="buoc-3-2"></a>

### Bước 3.2: Dịch địa chỉ bộ nhớ hai cấp (GPA → HVA)

<!-- ORIGINAL 148 END -->

<!-- ORIGINAL 149 BEGIN -->

**Phần cứng tham gia:** Cấu trúc bảng cây địa chỉ của QEMU (AddressSpaceDispatch).

<!-- ORIGINAL 149 END -->

<!-- ORIGINAL 151 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 151 END -->

<!-- ORIGINAL 153 BEGIN -->

- Dữ liệu payload được Guest khai báo tại địa chỉ GPA 0x00105000. QEMU (chạy ở Host User Space) không thể dùng con trỏ GPA này để truy cập bộ nhớ.

<!-- ORIGINAL 153 END -->

<!-- ORIGINAL 155 BEGIN -->

- QEMU tra cứu cấu trúc bảng vùng nhớ máy ảo để chuyển đổi:

<!-- ORIGINAL 155 END -->

<!-- ORIGINAL 157 BEGIN -->

- \{GPA \} (0x00105000) ----\> \\text\{Host Virtual Address (HVA) \} (0x7f11a000)\$\$

<!-- ORIGINAL 157 END -->

<!-- ORIGINAL 158 BEGIN -->

- Bản chất sao chép (Điểm mấu chốt):

<!-- ORIGINAL 158 END -->

<!-- ORIGINAL 160 BEGIN -->

- Vì cấu hình cache=none, QEMU không copy 4096 bytes này vào buffer riêng của nó. Con trỏ HVA 0x7f11a000 trỏ thẳng vào chính khung trang RAM vật lý mà Guest ban đầu đã chuẩn bị.

<!-- ORIGINAL 160 END -->

<!-- ORIGINAL 162 BEGIN -->

- Rủi ro copy ngầm: Chỉ khi Guest phát I/O bị phân mảnh (Scatter-Gather) mà các mảnh GPA ánh xạ thành các dải HVA không liên tục, QEMU mới buộc phải gọi qemu\_iovec\_to\_buf() để memcpy gom dữ liệu vào một bộ đệm phẳng tạm thời (Bounce Buffer).

<!-- ORIGINAL 162 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.2

```c
/* QEMU mapping API minh họa, không là syscall. */
void *hva = address_space_map(as, gpa, &mapped_len, is_write, attrs);
```

**Ý nghĩa và tham số:** `as` là AddressSpace của device; `gpa` là địa chỉ trong không gian device đang dùng; `mapped_len` là in/out số byte yêu cầu và số byte map được; `is_write` biểu thị backend sẽ ghi vào bộ nhớ Guest; `attrs` là thuộc tính giao dịch. Kết quả là HVA hoặc mapping tạm theo loại region. Với payload write xuống disk, backend đọc bộ nhớ; status lại là buffer backend ghi.

**Device và lưu ý:** GPA→HVA là tra mapping phần mềm QEMU, khác GVA→GPA→HPA qua CPU/EPT. HVA rời rạc có thể biểu diễn bằng iovec và pwritev, không tự động bắt buộc copy. cache=none cũng không tự bảo đảm mọi mapping/backend zero-copy. [V4][V6]

<!-- ORIGINAL 164 BEGIN -->

<a id="buoc-3-3"></a>

### Bước 3.3: Phát lời gọi hệ thống Linux Native AIO (io\_submit)

<!-- ORIGINAL 164 END -->

<!-- ORIGINAL 165 BEGIN -->

**Phần cứng tham gia:** Thanh ghi CPU Host x86-64.

<!-- ORIGINAL 165 END -->

<!-- ORIGINAL 167 BEGIN -->

Cơ chế & Dữ liệu:

<!-- ORIGINAL 167 END -->

<!-- ORIGINAL 169 BEGIN -->

- Nhờ cấu hình io=native, QEMU không dùng luồng POSIX AIO blocking. Nó chuẩn bị một cấu trúc mô tả I/O bất đồng bộ của Kernel: struct iocb.

<!-- ORIGINAL 169 END -->

<!-- ORIGINAL 171 BEGIN -->

- Trường iocb-\>aio\_buf được gán trực tiếp bằng con trỏ HVA: 0x7f11a000.

<!-- ORIGINAL 171 END -->

<!-- ORIGINAL 173 BEGIN -->

- QEMU phát chỉ lệnh syscall gọi hàm io\_submit():

<!-- ORIGINAL 173 END -->

<!-- ORIGINAL 175 BEGIN -->

- C

<!-- ORIGINAL 175 END -->

<!-- ORIGINAL 176 BEGIN -->

io\_submit(host\_aio\_ctx, 1, &iocb\_ptr);

<!-- ORIGINAL 176 END -->

<!-- ORIGINAL 179 BEGIN -->

- CPU Host chuyển đặc quyền từ Host Ring 3 (QEMU) sang Host Ring 0 (Kernel).

<!-- ORIGINAL 179 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 3.3

```c
/* Ví dụ libaio, chỉ dùng sau khi đã tạo ctx và mở fd phù hợp. */
struct iocb cb;
struct iocb *cbs[1] = { &cb };
io_prep_pwritev(&cb, host_fd, iov, iovcnt, offset);
int submitted = io_submit(ctx, 1, cbs);
```

**Ý nghĩa và tham số:** `cb` là mô tả AIO; `host_fd` mở backend Host; `iov` là mảng iovec có iov_base và iov_len; `iovcnt` là số phần tử; `offset` là byte offset. `ctx` là native AIO context; `1` là số iocb gửi; `cbs` là mảng con trỏ. libaio trả số request được nhận hoặc mã lỗi âm; chưa trả số byte đã ghi. Phải xử lý submit thiếu request và lỗi.

**Device và lưu ý:** Với IOCB_CMD_PWRITEV ở ABI kernel, aio_buf trỏ tới mảng iovec, aio_nbytes là số iovec; khác IOCB_CMD_PWRITE, nơi aio_buf trỏ payload và aio_nbytes là số byte. Không kết hợp opcode PWRITEV với các trường nghĩa PWRITE như sơ đồ gốc. libaio wrapper và raw syscall có quy ước lỗi khác nhau. [V7]

<!-- ORIGINAL 181 BEGIN -->

<a id="chang-4"></a>

## CHẶNG 4: HOST STORAGE STACK & MẠNG iSCSI

<!-- ORIGINAL 181 END -->

<!-- ORIGINAL 182 BEGIN -->

(Biến đổi: HVA → Host PA → SCSI CDB → iSCSI PDU → NIC DMA → Cáp quang)

<!-- ORIGINAL 182 END -->

<!-- ORIGINAL 184-201 BEGIN -->

```text
[ Host Kernel: blk-mq ]
  - Khóa trang RAM: HVA (0x7f11a000) -> Host PA (0x1a2b3000)
  - Cấp phát struct bio & struct request
       │
       ▼ (dm-multipath)
[ /dev/mapper/mpath0 -> Chọn /dev/sda ]
       │
       ▼ (SCSI Subsystem & iscsi_tcp)
[ SCSI CDB (WRITE_10) -> iSCSI Command PDU ]
  - SG-List chứa [Host PA: 0x1a2b3000, Len: 4096]
       │
       ▼ (Host TCP/IP Stack & IOMMU)
[ IOMMU dịch: Host PA (0x1a2b3000) -> IOVA (0x80001000) ]
  - Nạp IOVA vào TX Ring của Card mạng
  - CPU gõ Doorbell MMIO lên Card NIC
       │
       ▼ (Card NIC DMA)
[ NIC DMA Engine đọc RAM Host -> Bắn Frame quang sang NetApp Target ]
```

<!-- ORIGINAL 184-201 END -->

<!-- ORIGINAL 202 BEGIN -->

(Giai đoạn này diễn ra hoàn toàn tương đồng với Chặng 2, 3, 4, 5 của Luồng Host-write):

<!-- ORIGINAL 202 END -->

<!-- ORIGINAL 204 BEGIN -->

- Dịch HVA → Host PA: Host Kernel nhận HVA 0x7f11a000, gọi get\_user\_pages() để khóa trang RAM vật lý của Host (Host PA 0x1a2b3000) vào struct bio.

<!-- ORIGINAL 204 END -->

<!-- ORIGINAL 206 BEGIN -->

- Qua Multipath & SCSI: dm-multipath clone request sang /dev/sda. Tầng SCSI dịch thành WRITE\_10 CDB.

<!-- ORIGINAL 206 END -->

<!-- ORIGINAL 208 BEGIN -->

- iSCSI & Mạng: Driver iscsi\_tcp bọc lệnh vào PDU, đưa vào sk\_buff của socket TCP.

<!-- ORIGINAL 208 END -->

<!-- ORIGINAL 210 BEGIN -->

- IOMMU & NIC DMA: IOMMU dịch Host PA 0x1a2b3000 thành IOVA 0x80001000. Card NIC Host dùng DMA Engine rút 4096 bytes dữ liệu từ thanh RAM Host đẩy qua cáp quang tới tủ NetApp.

<!-- ORIGINAL 210 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 4.1

```c
/* Bổ sung cho dòng Dịch HVA → Host PA của bản gốc. */
long nr = pin_user_pages_fast(hva, nr_pages, flags, host_pages);
submit_bio(host_bio);
```

**Ý nghĩa và tham số:** `hva` thuộc tiến trình QEMU trên Host; `host_pages` nhận trang Host; nr_pages/flags có nghĩa tương tự API pin nhưng ở một kernel khác. `host_bio` mô tả sector và vector của backend Host. Direct I/O thực tế có thể gọi helper khác để lấy trang từ iterator.

**Device và lưu ý:** Guest pin và Host pin có lifetime riêng. HPA không phải số fd, offset file hay LBA. Payload chỉ dùng cùng trang vật lý khi mapping RAM và backend thực tế cho phép. [V3]


#### Bổ sung lời gọi hàm và phân tích tại bước 4.2

```c
submit_bio(host_bio);
/* Symbol SCSI disk thường gặp, cần đối chiếu source kernel: */
blk_status_t sd_init_command(struct scsi_cmnd *cmd);
```

**Ý nghĩa và tham số:** `host_bio` hướng tới device logic; Device Mapper chọn path và remap bio/request theo mode. `cmd` là SCSI command được sd driver chuẩn bị CDB cùng độ dài/hướng truyền. Request-based multipath có clone request; bio-based dùng đường map khác.

**Device và lưu ý:** `/dev/mapper/mpath0` và các `/dev/sdX` ở đây thuộc Host. SCSI WRITE(10)/(16) tính transfer length theo logical block size, không luôn 8 block cho 4096 byte. [V8][V9]


#### Bổ sung lời gọi hàm và phân tích tại bước 4.3

```c
/* Điểm vào SCSI transport thường gặp, không phải call stack đã trace. */
iscsi_queuecommand(host, sc);
/* Socket interface minh họa ở tầng sau: */
sock_sendmsg(sock, &msg);
```

**Ý nghĩa và tham số:** `host` là SCSI host logic; `sc` là lệnh có CDB/SG. Libiscsi tạo task/ITT và PDU theo thông số session. `sock` là socket TCP; `msg` chứa iterator và cờ gửi; giá trị trả là byte được đường gửi nhận hoặc lỗi, không là persistence.

**Device và lưu ý:** iSCSI command có thể kèm Immediate Data hoặc có Data-Out/R2T. TCP segment không tương ứng một-một với iSCSI PDU. Copy hoặc giữ tham chiếu trang phụ thuộc transport và network path. [V10]


#### Bổ sung lời gọi hàm và phân tích tại bước 4.4

```c
dma_addr_t dma = dma_map_page(nic_dev, page, off, len, DMA_TO_DEVICE);
/* Chỉ dùng khi dma_mapping_error(nic_dev, dma) là false. */
/* Sau điền TX descriptor và ordering theo driver: */
writel(tail, doorbell);
```

**Ý nghĩa và tham số:** `nic_dev` là NIC; `page`, `off`, `len` xác định vùng RAM; hướng DMA_TO_DEVICE cho NIC đọc. `dma` là địa chỉ DMA. `tail` và `doorbell` phụ thuộc NIC. NIC tiếp tục DMA và MAC/PHY phát tín hiệu mà không có hàm CPU riêng cho từng byte.

**Device và lưu ý:** IOMMU ánh xạ địa chỉ DMA của NIC tới HPA; không đồng nhất virtual IOMMU Guest, EPT và IOMMU vật lý. Network driver có thể dùng mapping khác, bounce hoặc tái sử dụng mapping; không suy ra 0 CPU cho toàn I/O. [V11]

<!-- ORIGINAL 212 BEGIN -->

<a id="chang-5"></a>

## CHẶNG 5: CHIỀU HOÀN TẤT VÀ VÒNG LẶP TIÊM NGẮT ẢO (COMPLETION PATH)

<!-- ORIGINAL 212 END -->

<!-- ORIGINAL 213 BEGIN -->

(Biến đổi: Phản hồi mạng → Ngắt phần cứng Host → Đánh thức QEMU → Tiêm ngắt ảo vào VM)

<!-- ORIGINAL 213 END -->

<!-- ORIGINAL 216-242 BEGIN -->

```text
[ NetApp Target gửi iSCSI Response OK ]
       │
       ▼ (Cáp mạng truyền về Host)
[ Card NIC Host: Bắn ngắt phần cứng MSI-X ]
       │
       ▼ (CPU Host chạy Hard-IRQ -> SoftIRQ)
[ Host Kernel: TCP Stack -> iscsi_tcp -> blk-mq ]
  - Host AIO đánh dấu iocb hoàn tất
  - Ghi tín hiệu vào eventfd liên kết với QEMU
       │
       ▼ (ĐIỂM NGHẼN 4: CONTEXT SWITCH LẦN 2)
[ QEMU IOThread thức dậy ]
  - Gọi io_getevents() nhận kết quả
  - Cập nhật Virtqueue:
    + Ghi 0 (VIRTIO_BLK_S_OK) vào Status byte trên RAM Guest
    + Ghi chỉ số Desc vào Used Ring (used->ring[idx])
    + Tăng used->idx
  - Gõ chuông báo KVM: Ghi vào irqfd
       │
       ▼ (ĐIỂM NGHẼN 5: TIÊM NGẮT ẢO - VIRTUAL IRQ)
[ Module KVM ]
  - KVM nhận irqfd -> Tiêm Virtual Interrupt vào vCPU của Guest
       │
       ▼ (Guest vCPU bị ngắt)
[ Guest Kernel: Driver virtio-blk ]
  - Chạy hàm phục vụ ngắt ISR của virtio-blk
  - Đánh thức luồng ứng dụng fio (Guest Ring 3) kết thúc lệnh write()
```

<!-- ORIGINAL 216-242 END -->

<!-- ORIGINAL 244 BEGIN -->

<a id="buoc-5-1"></a>

### Bước 5.1: Mạng Host nhận kết quả và ngắt phần cứng

<!-- ORIGINAL 244 END -->

<!-- ORIGINAL 245 BEGIN -->

- Target xác nhận ghi xong, gửi iSCSI Response PDU.

<!-- ORIGINAL 245 END -->

<!-- ORIGINAL 247 BEGIN -->

- NIC Host nhận gói tin, kích hoạt ngắt MSI-X lên một CPU Core của Host.

<!-- ORIGINAL 247 END -->

<!-- ORIGINAL 249 BEGIN -->

- Ngắt mềm NET\_RX\_SOFTIRQ bóc gói TCP, iscsi\_tcp khớp ITT, blk-mq báo hoàn tất cho Linux Native AIO.

<!-- ORIGINAL 249 END -->

<!-- ORIGINAL 251 BEGIN -->

- Kernel Host gửi tín hiệu hoàn thành thông qua cơ chế eventfd của AIO.

<!-- ORIGINAL 251 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.1

```c
napi_schedule(&rx_queue->napi);
/* Sau khi storage stack đã nhận kết quả: */
blk_mq_complete_request(host_rq);
```

**Ý nghĩa và tham số:** `rx_queue->napi` là NAPI instance của NIC. Poll xử lý packet trong ngân sách; TCP chuyển byte đã reassemble cho iSCSI, ITT ghép task. `host_rq` là request Host cần completion. AIO tạo kết quả và có thể signal eventfd nếu iocb được cấu hình thông báo.

**Device và lưu ý:** TCP ACK không thay SCSI GOOD; GOOD không luôn nghĩa đã ghi vào NAND. Ngắt có thể được gom, NAPI có thể poll, và AIO eventfd chỉ dùng khi đăng ký/cấu hình. [V10][V12]

<!-- ORIGINAL 253 BEGIN -->

<a id="buoc-5-2"></a>

### Bước 5.2: QEMU IOThread nhận hoàn tất & Cập nhật Used Ring

<!-- ORIGINAL 253 END -->

<!-- ORIGINAL 254 BEGIN -->

- Luồng QEMU IOThread thức dậy (lại một đợt Context Switch trên Host), gọi io\_getevents() thu hoạch kết quả.

<!-- ORIGINAL 254 END -->

<!-- ORIGINAL 256 BEGIN -->

- Cập nhật Virtqueue:

<!-- ORIGINAL 256 END -->

<!-- ORIGINAL 258 BEGIN -->

- QEMU ghi giá trị 0 (VIRTIO\_BLK\_S\_OK) vào ô nhớ Status byte (Desc 2) trên RAM của Guest thông qua con trỏ HVA tương ứng.

<!-- ORIGINAL 258 END -->

<!-- ORIGINAL 260 BEGIN -->

- QEMU nạp chỉ số Descriptor vào Used Ring (used-\>ring) và tăng biến đếm used-\>idx.

<!-- ORIGINAL 260 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.2

```c
/* Ví dụ user API libaio; QEMU có thể đọc completion ring tối ưu hơn. */
int nev = io_getevents(ctx, min_nr, max_nr, events, timeout);
/* Backend virtio completion minh họa: */
virtqueue_push(vq, elem, used_len);
```

**Ý nghĩa và tham số:** `ctx` là AIO context; min_nr/max_nr đặt số event; events nhận io_event; timeout giới hạn chờ. Kết quả trả là số event; `events[i].res` chứa số byte hoặc lỗi của I/O. `vq`, `elem` xác định request virtio cần trả; `used_len` là số byte device đã ghi vào các buffer in, không phải mặc nhiên 4096 byte payload write.

**Device và lưu ý:** Ghi status/used ring phải theo thứ tự memory visibility phù hợp trước notify. QEMU có đường thu hoạch ring trực tiếp và polling, nên io_getevents/context switch trong bản gốc không là lịch trình cố định cho mỗi request. [V4][V13]

<!-- ORIGINAL 262 BEGIN -->

<a id="buoc-5-3"></a>

### Bước 5.3: Báo hiệu qua irqfd và KVM tiêm ngắt ảo

<!-- ORIGINAL 262 END -->

<!-- ORIGINAL 263 BEGIN -->

- Để báo cho Guest VM biết I/O đã xong, QEMU ghi 8-byte vào irqfd.

<!-- ORIGINAL 263 END -->

<!-- ORIGINAL 265 BEGIN -->

- KVM can thiệp: KVM nhận sự kiện từ irqfd, truy cập vào bộ điều khiển ngắt ảo (vAPIC) của vCPU và tiêm một ngắt ảo (Virtual Interrupt Injection) vào máy ảo.

<!-- ORIGINAL 265 END -->

<!-- ORIGINAL 267 BEGIN -->

- Nếu CPU hỗ trợ tính năng ảo hóa cao cấp (Intel APIC-v / Posted Interrupts), KVM có thể bắn ngắt thẳng vào vCPU mà không bắt vCPU phải thoát (Zero VM-Exit). Nếu không, vCPU bị gián đoạn và phải đổi ngữ cảnh để xử lý ngắt.

<!-- ORIGINAL 267 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.3

```c
/* Đăng ký irqfd ở control path. */
ioctl(vm_fd, KVM_IRQFD, &irq_registration);
/* Thông báo user space qua eventfd khi route dùng irqfd: */
uint64_t one = 1;
write(irq_eventfd, &one, sizeof(one));
```

**Ý nghĩa và tham số:** `irq_registration` chỉ eventfd và guest interrupt route; `vm_fd` là VM descriptor. `irq_eventfd` là eventfd đã đăng ký; write số 1 tăng counter. KVM chuyển sự kiện thành guest interrupt theo routing. Những lời gọi đăng ký không lặp lại cho mỗi I/O.

**Device và lưu ý:** ioeventfd báo Guest→backend, irqfd báo backend→Guest. Phải có route/khả năng tương ứng; một số cấu hình dùng cơ chế notify khác. Không mặc định mỗi completion cần một IRQ hoặc một VM-Exit; batching, suppression, APIC virtualization và polling làm thay đổi số sự kiện. [V5]

<!-- ORIGINAL 269 BEGIN -->

<a id="buoc-5-4"></a>

### Bước 5.4: Guest kết thúc chu trình

<!-- ORIGINAL 269 END -->

<!-- ORIGINAL 270 BEGIN -->

- Driver virtio-blk trong Guest nhận ngắt ảo, chạy trình phục vụ ngắt (ISR), thu hồi các Descriptor trong Virtqueue.

<!-- ORIGINAL 270 END -->

<!-- ORIGINAL 272 BEGIN -->

- Block layer của Guest báo bio hoàn tất, đưa tiến trình fio trong Guest về trạng thái sẵn sàng chạy (TASK\_RUNNING).

<!-- ORIGINAL 272 END -->

<!-- ORIGINAL 274 BEGIN -->

- Lời gọi hàm write() trong Guest VM chính thức hoàn tất.

<!-- ORIGINAL 274 END -->


#### Bổ sung lời gọi hàm và phân tích tại bước 5.4

```c
/* API virtqueue thu hồi completion trong Guest. */
void *token = virtqueue_get_buf(vq, &used_len);
/* Sau khi kiểm tra status của request Guest: */
blk_mq_end_request(guest_rq, status);
```

**Ý nghĩa và tham số:** `vq` thuộc driver Guest; used_len nhận số byte device ghi; token liên hệ request đã enqueue. `guest_rq` là request Guest, status là blk_status_t được chuyển từ status virtio. Driver thu hồi descriptor, hoàn tất block I/O và trả lên syscall/application.

**Device và lưu ý:** VIRTIO_BLK_S_OK là byte status device, không phải giá trị trả write. write đồng bộ trả số byte hoặc lỗi; AIO/IO_uring Guest có cơ chế nhận completion riêng. Guest còn phải xử lý trạng thái lỗi, tháo pin và lifetime buffer phù hợp. [V4]

<!-- ORIGINAL 276 BEGIN -->

<a id="chang-6"></a>

## CHẶNG 6: TỔNG KẾT VI PHẪU RANH GIỚI ẢO HÓA (OVERHEAD BREAKDOWN)

<!-- ORIGINAL 276 END -->

<!-- ORIGINAL 277 BEGIN -->

Từ vi phẫu trên, ta bóc tách định lượng chính xác 5 điểm nghẽn vật lý làm cho luồng VM-write chậm hơn và có Tail Latency (P99) cao hơn hẳn so với Host-write:

<!-- ORIGINAL 277 END -->

<!-- ORIGINAL 279 BEGIN -->

<a id="tong-ket-1"></a>

### 1. Bản đồ biến đổi địa chỉ 5 cấp (The 5-Level Address Chain)

<!-- ORIGINAL 279 END -->

<!-- ORIGINAL 280-292 BEGIN -->

```text
[1. Ứng dụng Guest]    Guest Virtual Address (GVA: 0x00401000)
         │             ├── MMU Guest dịch qua Guest Page Table
         ▼
[2. Kernel Guest]      Guest Physical Address (GPA: 0x00105000)
         │             ├── QEMU dịch qua MemoryRegion (mmap)
         ▼
[3. QEMU Process]      Host Virtual Address (HVA: 0x7f11a000)
         │             ├── Kernel Host dịch & pin qua get_user_pages
         ▼
[4. Kernel Host]       Host Physical Address (HPA: 0x1a2b3000)
         │             ├── IOMMU phần cứng dịch qua bảng trang VT-d
         ▼
[5. Phần cứng NIC]     I/O Virtual Address (IOVA: 0x80001000 trên bus PCIe)
```

<!-- ORIGINAL 280-292 END -->

<!-- ORIGINAL 293 BEGIN -->

Mỗi cấp chuyển đổi đòi hỏi việc duy trì và đồng bộ bảng trang bộ nhớ (GPT, EPT, Host Page Table, IOMMU Page Table), làm tăng áp lực khủng khiếp lên bộ nhớ đệm dịch địa chỉ của CPU (TLB - Translation Lookaside Buffer), dẫn đến hiện tượng TLB Miss liên tục.

<!-- ORIGINAL 293 END -->

<!-- ORIGINAL 295 BEGIN -->

<a id="tong-ket-2"></a>

### 2. Định lượng các cú nhảy ngữ cảnh & Bẫy CPU (Context Switches & Traps)

<!-- ORIGINAL 295 END -->

<!-- ORIGINAL 296 BEGIN -->

| Chỉ số vi kiến trúc | Luồng 1: Host-write | Luồng 2: VM-write | Phần chênh lệch (Virtualization Overhead) |
| --- | --- | --- | --- |
| Bẫy phần cứng VM-Exit | 0 | Ít nhất 1 lần / Batch I/O | Mất ∼1.5-2.5μ s CPU bị "đóng băng" tại cửa Doorbell MMIO. |
| Số lần Context Switch (Đổi luồng) | 1 (Khi app chờ I/O) | Ít nhất 3 lần | 1 lần đổi sang QEMU IOThread + 1 lần đổi trả về vCPU + 1 lần đổi khi nhận AIO completion. |
| Số lần Privilege Switch (Ring) | 2 (Ring 3 ↔ Ring 0) | 6 lần | Guest (Ring 3 → 0) → VM-Exit → QEMU (Host Ring 3) → Syscall (Host Ring 0) → Trả về. |
| Cơ chế IPC trung gian | Không có | Có (ioeventfd, irqfd) | Đẩy gói tin qua lại giữa Kernel KVM và tiến trình User-Space QEMU. |

<!-- ORIGINAL 296 END -->

<!-- ORIGINAL 297 BEGIN -->

<a id="tong-ket-3"></a>

### 3. Phân định rạch ròi số lần memcpy trong Luồng VM-write

<!-- ORIGINAL 297 END -->

<!-- ORIGINAL 298 BEGIN -->

Báo cáo cần kết luận chính xác về việc sao chép bộ nhớ:

<!-- ORIGINAL 298 END -->

<!-- ORIGINAL 300 BEGIN -->

- Payload memcpy (Lý tưởng): 0 LẦN. Nhờ cấu hình cache=none và O\_DIRECT, QEMU truyền con trỏ HVA ánh xạ từ GPA xuống Kernel Host, và NIC DMA trực tiếp từ RAM vật lý của Host.

<!-- ORIGINAL 300 END -->

<!-- ORIGINAL 301 BEGIN -->

- Payload memcpy (Thực tế rủi ro): Có thể phát sinh 1 LẦN nếu:

<!-- ORIGINAL 301 END -->

<!-- ORIGINAL 302 BEGIN -->

- Bộ đệm trong Guest không căn chuẩn 512B/4KiB.

<!-- ORIGINAL 302 END -->

<!-- ORIGINAL 303 BEGIN -->

- Các trang GPA trong Virtqueue bị phân tán thành nhiều mảnh nhỏ không liên tục trên không gian HVA của Host, buộc QEMU phải gọi qemu\_iovec\_to\_buf() để tạo Bounce Buffer.

<!-- ORIGINAL 303 END -->

<!-- ORIGINAL 305 BEGIN -->

- Metadata copy: GẤP ĐÔI so với Host-write. CPU phải khởi tạo và sao chép:  


<!-- ORIGINAL 305 END -->

<!-- ORIGINAL 306 BEGIN -->

- \{Guest bio\} ----\> \\text\{Virtq Descriptor\} ----\> \\text\{QEMU iocb\} ----\> \\text\{Host bio\} ----\> \\text\{Host request\} ----\> \\text\{SCSI CDB\} ----\> \\text\{iSCSI PDU\}\$\$

<!-- ORIGINAL 306 END -->

<!-- ORIGINAL 307 BEGIN -->

<a id="ket-luan-goc"></a>

## KẾT LUẬN:

<!-- ORIGINAL 307 END -->

<!-- ORIGINAL 308 BEGIN -->

Điểm làm cho kiến trúc ảo hóa truyền thống chậm chạp và làm biến động Tail Latency (P99) không phải do CPU bốc vác 4 KiB payload, mà là do "Thuế ảo hóa" (Virtualization Tax):

<!-- ORIGINAL 308 END -->

<!-- ORIGINAL 310 BEGIN -->

- Chi phí dừng vi xử lý của VM-Exit tại ranh giới Doorbell.

<!-- ORIGINAL 310 END -->

<!-- ORIGINAL 312 BEGIN -->

- Sự phân mảnh hàng đợi: I/O phải đi qua hai lần Block Layer (Guest blk-mq rồi lại đến Host blk-mq).

<!-- ORIGINAL 312 END -->

<!-- ORIGINAL 314 BEGIN -->

- Độ trễ đánh thức luồng: Tiến trình QEMU IOThread phải thức dậy qua ioeventfd, gọi Syscall rồi lại nhận kết quả qua eventfd để tiêm ngắt ảo irqfd.

<!-- ORIGINAL 314 END -->

<!-- ORIGINAL 316 BEGIN -->

Đây chính là lý do vì sao SPDK vhost-user-blk ra đời: SPDK dùng Polling để triệt tiêu 100% VM-Exit, dùng Shared Hugepages để triệt tiêu QEMU IOThread, và bắn thẳng xuống NVMe-oF để bỏ qua toàn bộ Host Block Layer và iSCSI/TCP stack.

<!-- ORIGINAL 316 END -->


<a id="device-ao"></a>

## Bổ sung device và các không gian địa chỉ

| Thành phần | Quan sát từ đâu | Vai trò |
| --- | --- | --- |
| `/dev/vdb` | Guest | Device block ảo do virtio-blk cung cấp; nơi ứng dụng Guest gửi I/O. Không phải tên path iSCSI Host. |
| `/dev/sdX` với virtio-scsi | Guest | Device SCSI ảo; chữ sd không chứng minh đang truy cập disk vật lý trực tiếp. |
| virtio-blk PCI hoặc MMIO device | Guest | Device ảo có configuration, queue và notify; RAM chứa virtqueue khác vùng register ảo. |
| Virtqueue | Guest và backend | Cấu trúc giao việc/hoàn tất trong RAM, không phải node `/dev` và không phải ổ lưu trữ riêng. |
| `/dev/mapper/mpath0` | Host | Device block logic hợp nhất các path tới cùng LUN. |
| `/dev/dm-N` | Host | Node Device Mapper mà alias multipath có thể trỏ tới. N không cố định. |
| `/dev/sda`, `/dev/sdb` | Host trong ví dụ | Các SCSI path có thể cùng trỏ tới một remote LUN; không tự suy ra hai disk độc lập. |
| LUN | Storage target | Device lưu trữ logic target xuất ra; backend thực có thể là volume/pool hoặc cấu trúc khác. |
| `/dev/kvm` và VM/vCPU fd | Host | Giao diện điều khiển/running KVM; không chứa payload của volume. |
| ioeventfd | Host | Báo queue notify từ Guest tới backend khi được đăng ký. |
| AIO completion eventfd | Host | Thông báo native AIO hoàn tất nếu đã cấu hình; khác ioeventfd và irqfd. |
| irqfd | Host và KVM | Eventfd gắn guest interrupt route để báo Guest khi được cấu hình. |
| `/dev/vhost-net` | Host | Backend vhost kernel dành cho mạng; không chứng minh virtio-blk disk dùng vhost. |
| `/dev/vhost-scsi` | Host | Backend vhost kernel dành cho SCSI nếu cấu hình; khác virtio-blk QEMU thông thường. |
| Socket vhost-user-blk | Host | Kênh điều khiển QEMU–backend user space, không phải block node Guest. |

**Các tên `/dev` có tính cục bộ theo hệ điều hành.** `/dev/vdb` của Guest và `/dev/mapper/mpath0` của Host nối qua cấu hình backing của QEMU, không qua quy tắc tên giống nhau. Một disk Guest cũng có thể dùng image file, local NVMe, backend network hoặc SPDK. Để xác nhận, đọc XML libvirt/QEMU block graph và mapping device thật.

| Loại địa chỉ | Ai dùng | Ý nghĩa |
| --- | --- | --- |
| GVA | Tiến trình Guest | Địa chỉ ảo của buffer Guest. |
| GPA | Guest kernel và mô hình device | Địa chỉ vật lý theo góc nhìn Guest; có thể khác địa chỉ virtio DMA khi dùng virtual IOMMU. |
| HVA | QEMU hoặc backend Host | Địa chỉ ảo Host ánh xạ RAM Guest; sử dụng trong iovec/native AIO. |
| HPA | Host kernel và bộ nhớ vật lý | Địa chỉ vật lý Host chứa dữ liệu thực. |
| IOVA/DMA address NIC | NIC và IOMMU vật lý | Địa chỉ NIC dùng đọc/ghi RAM, do Host DMA mapping thiết lập. |
| File offset hoặc LBA | Filesystem và storage device | Vị trí lưu dữ liệu trên device, không phải địa chỉ RAM. |

Đây là các cách nhìn địa chỉ của cùng dữ liệu trong trường hợp mapping trực tiếp; không phải năm bản sao payload. EPT/NPT phục vụ truy cập CPU của vCPU; mapping GPA→HVA phục vụ backend phần mềm; IOMMU/DMA mapping phục vụ NIC. Các bước không nhất thiết là năm lần tra bảng tuần tự xảy ra cho mọi byte.

**Virtio và vhost:** Virtio định nghĩa giao diện device/queue; QEMU, vhost kernel và vhost-user là các cách triển khai backend khác nhau. Trong luồng nguyên văn, backend block là QEMU IOThread dùng Host native AIO. Không tự thêm vhost-net vào luồng disk. Với SPDK vhost-user-blk, QEMU trao RAM và thông tin queue cho backend qua vhost-user; SPDK xử lý queue và gửi I/O xuống bdev thích hợp. [V14][V15]

**Lời gọi cấu hình SPDK minh họa:** RPC `vhost_create_blk_controller(ctrlr, dev_name, cpumask)` tạo controller block tên `ctrlr`, gắn bdev `dev_name`, với mask CPU khi được hỗ trợ. Cần đối chiếu schema RPC đúng bản SPDK; đây là control path, không phải hàm thực hiện mỗi write. Đường mỗi I/O nằm ở xử lý virtqueue → bdev → driver/backend, có thể là NVMe-oF/RDMA nếu cấu hình. [V15]

<a id="dieu-kien-overhead"></a>

## Bổ sung điều kiện của các kết luận overhead

Phần này làm rõ điều kiện áp dụng của nguyên văn phía trên. Các con số và nhận định gốc vẫn được giữ nguyên để đối chiếu.

**Số lần chuyển chế độ và độ trễ phải đo.** Các giá trị 1.5–2.5 μs, ít nhất 3 context switch và 6 privilege switch trong bản gốc chưa kèm dữ liệu đo hoặc cấu hình. Không dùng chúng như hằng số kiến trúc. Batching, notify suppression, polling, CPU affinity, scheduler, interrupt coalescing và backend thay đổi số sự kiện trên mỗi request. VM-Exit, syscall privilege switch, context switch và guest IRQ là các sự kiện khác nhau.

**Iovec không liên tục chưa phải lý do bắt buộc copy.** Native AIO PWRITEV nhận nhiều iovec; các HVA rời rạc có thể được truyền qua vector. Copy/bounce còn phụ thuộc alignment, giới hạn I/O, kiểu RAM region và đường backend. Với ABI PWRITEV, `aio_buf` là địa chỉ mảng iovec và `aio_nbytes` là số phần tử, khác ý nghĩa của PWRITE. [V7]

**Cache và memcpy là hai vấn đề khác nhau.** Guest O_DIRECT và Host cache=none giúp tránh page cache tương ứng; không tự chứng minh zero-copy toàn đường. Mô hình 0 payload copy là khả năng của đường tối ưu, cần trace để xác nhận. Metadata nhiều hơn không đồng nghĩa đúng gấp đôi: phải xác định đếm byte, lần sao chép, cấu trúc được tạo hay CPU cycles. [V1][V6]

**TLB miss không xảy ra liên tục chỉ vì có nhiều không gian địa chỉ.** TLB/cache bảng trang, hugepage, locality và kích thước working set quyết định hành vi. QEMU tra MemoryRegion là mapping phần mềm, không phải một cấp page table CPU. Muốn kết luận phải đo đúng counter/sự kiện của CPU và kết hợp trace.

**SPDK polling không bảo đảm loại bỏ 100% VM-Exit.** Polling có thể giảm nhu cầu notify queue và wakeup trong datapath; VM vẫn có thể exit vì timer, emulation và nhiều nguyên nhân khác. Hugepage hỗ trợ quản lý RAM/địa chỉ cho backend, không tự nó loại bỏ QEMU IOThread. Vhost-user chuyển xử lý queue sang backend khác; QEMU vẫn có vai trò device/control plane. [V14][V15]

**Bỏ Host block layer và iSCSI/TCP phụ thuộc backend.** Đường SPDK bdev NVMe-oF/RDMA có thể tránh kernel block/iSCSI/TCP datapath đang xét; SPDK bdev dùng kernel backend hoặc NVMe/TCP có đường khác. Không suy ra mọi cấu hình SPDK đều bỏ TCP hay đều dùng RDMA. [V15]

**Cách chốt báo cáo trên lab:** Ghi phiên bản Guest/Host kernel và QEMU, virtio transport/ring type, IOThread mapping, AIO engine, cache mode, Device Mapper mode, target/cache và NIC. Theo dõi timeline request Guest → virtqueue → backend submit → Host block/network → Host completion → used ring → Guest completion. Dùng KVM exit events, scheduler events, block trace và thông tin backend để đếm overhead, đồng thời xác định buffer lifetime/copy. Một request có thể chia hoặc một batch có thể gồm nhiều request; phải ghi rõ đơn vị thống kê.

<a id="nguon-tham-khao"></a>

## Nguồn tham khảo cho nội dung bổ sung



- **[V1]** [Linux open và O_DIRECT](https://man7.org/linux/man-pages/man2/open.2.html).
- **[V2]** [Linux posix_memalign](https://man7.org/linux/man-pages/man3/posix_memalign.3.html).
- **[V3]** [Linux pin_user_pages](https://kernel.org/doc/html/latest/core-api/pin_user_pages.html).
- **[V4]** [OASIS VIRTIO Version 1.2](https://docs.oasis-open.org/virtio/virtio/v1.2/virtio-v1.2.html).
- **[V5]** [Linux 6.8 KVM API](https://www.kernel.org/doc/html/v6.8/virt/kvm/api.html).
- **[V6]** [QEMU invocation và cấu hình disk](https://www.qemu.org/docs/master/system/invocation.html).
- **[V7]** [Linux io_submit và ABI iocb](https://man7.org/linux/man-pages/man2/io_submit.2.html).
- **[V8]** [Red Hat Multipath Devices](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux/7/html/dm_multipath/mpath_devices).
- **[V9]** [Linux 6.8 SCSI mid low level API](https://www.kernel.org/doc/html/v6.8/scsi/scsi_mid_low_api.html).
- **[V10]** [RFC 7143 iSCSI Protocol](https://www.rfc-editor.org/rfc/rfc7143.html).
- **[V11]** [Linux 6.8 Dynamic DMA mapping Guide](https://www.kernel.org/doc/html/v6.8/core-api/dma-api-howto.html).
- **[V12]** [Linux 6.8 NAPI](https://www.kernel.org/doc/html/v6.8/networking/napi.html).
- **[V13]** [Linux io_getevents](https://man7.org/linux/man-pages/man2/io_getevents.2.html).
- **[V14]** [QEMU Vhost user Protocol](https://www.qemu.org/docs/master/interop/vhost-user.html).
- **[V15]** [SPDK vhost Target](https://spdk.io/doc/vhost.html).
