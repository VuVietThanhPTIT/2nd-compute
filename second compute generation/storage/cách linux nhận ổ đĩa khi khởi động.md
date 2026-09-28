```
[1. Bật máy — Firmware quét bus]
        ↓
[2. Kernel Linux khởi động — nạp driver]
        ↓
[3. Kernel "enumerate" thiết bị — dò tìm cái gì cắm ở đâu]
        ↓
[4. Kernel tạo ra device node (/dev/sdX, /dev/nvme0n1)]
        ↓
[5. udev đặt tên ổn định, tạo symlink]
        ↓
[6. Ứng dụng gọi read()/write() → đi qua Block Layer]
        ↓
[7. Block Layer → Driver → giao thức → phần cứng]
        ↓
[8. Phần cứng trả kết quả → ngắt (interrupt) → kernel xử lý xong]
```
1 , firmware quét bus 
- các firmware( uefi / bios ) quét các bus trên bo mạch chủ  , firmware hỏi từng khe có thiết bị gì và là loại gì 
	- PCIe ( Kết nối từ các thiết bị ngoại vi đến bo mạch chủ đến CPU) Trước đó loại cũ là PCI dùng chung 1 lane , giờ khác lane  
2 , nạp driver 
- Sau khi bootloader nạp kernel vào ram , kernel bắt đầu tự dò lại toàn bộ bus PCIe ( dùng thông tin mà firmware đã dò qua bảng ACPI ) tìm driver phù hợp với thiết bị ngoại vi đó ( cụ thể ở đây là )
	 - 1. **Tầng 1 — Kernel tìm driver cho cái gì đang thật sự nằm trên PCIe:**
	    - Với NVMe → tìm thấy chính ổ đĩa → gán driver `nvme`.
	    - Với SATA → tìm thấy chip AHCI (không phải ổ đĩa) → gán driver `ahci`.
	 - 2 Tầng 2 — Driver đó tự đi "hỏi tiếp" xem có ổ đĩa nào đằng sau nó:**
	    - Driver `ahci` sau khi nhận diện xong, tự hỏi từng cổng SATA của nó: "cổng này có ổ cắm vào không?"
	    - Driver `nvme` thì không cần bước này với ổ đĩa (vì ổ = controller = đã có ở tầng 1), nhưng vẫn hỏi tiếp bên trong ổ đó: "mày chia làm bao nhiêu namespace?"
		
		```
			        "Ổ đĩa" (thiết bị lưu trữ vật lý)
                    /                             \
            HDD (đĩa từ, cơ khí)         SSD (NAND flash, điện tử)
                    │                              │
        Gần như luôn dùng SATA        Có thể dùng SATA HOẶC NVMe
        (vì cơ khí đã là điểm nghẽn,   (SATA nếu cần tương thích máy cũ/rẻ,
         không cần giao thức nhanh)     NVMe nếu cần tốc độ cao)
		```
		- Nvme or sata là 1 cách nói chuyện của máy tính với thiết bị lưu trữ 
3,  Kernel enumerate thiết bị lưu trữ cụ thể

- **SATA/AHCI:** driver AHCI hỏi từng cổng (port) xem có ổ nào cắm không.
- **SCSI/SAS:** phức tạp hơn —   có khái niệm phân lớp: **HBA (Host Bus Adapter)** → **SCSI transport** → hỏi từng target → mỗi target trả lời có bao nhiêu **LUN** (Logical Unit Number, bạn đã gặp từ này ở iSCSI — đúng là cùng khái niệm gốc từ SCSI).
- **NVMe:** driver NVMe hỏi controller "mày có bao nhiêu **namespace**?" (namespace trong NVMe gần giống LUN trong SCSI — 1 controller vật lý có thể chia thành nhiều namespace logic).
- **iSCSI:** phần "phát hiện" không tự động lúc boot như 2 loại trên — cần chủ động chạy `iscsiadm discovery` + `login` (như bạn đã lab), vì thiết bị không cắm vật lý mà ở xa qua mạng.

 4,  Kernel tạo device node

Sau khi enumerate xong, kernel tạo ra 1 "cổng giao tiếp" trong `/dev` cho mỗi ổ tìm được — đây chính là phần đã nói ở câu hỏi trước (`/dev/sda`, `/dev/nvme0n1`). Tên `sdX` do driver SCSI/SATA subsystem cấp theo thứ tự phát hiện; tên `nvme0n1` do driver NVMe cấp (số đầu là controller, `n1` là namespace số 1).

5 ,  udev đặt tên ổn định

Vì tên `sdX`/`nvmeXnY` phụ thuộc thứ tự phát hiện (không ổn định — đã giải thích ở câu hỏi trước về `/dev/disk/by-path/`), có 1 tiến trình userspace tên **udev** chạy nền, lắng nghe sự kiện kernel gửi ra mỗi khi có thiết bị mới (qua **netlink**, 1 kênh giao tiếp kernel↔userspace), rồi áp các rule để tạo thêm symlink ổn định: `/dev/disk/by-id/`, `/dev/disk/by-path/`, `/dev/disk/by-uuid/`. Đây cũng chính là cơ chế xử lý **hotplug** (cắm nóng USB, ổ NVMe external) — kernel phát hiện phần cứng mới bất kỳ lúc nào, không chỉ lúc boot, và udev phản ứng lại ngay lập tức.
- Tóm lại là **symlink giải quyết vấn đề thay đổi tên sau môi lần khởi động lại**
#### 6-7. Ứng dụng đọc/ghi thật sự đi qua đâu

```
Ứng dụng gọi read()/write()
        ↓
VFS (Virtual File System) — lớp trừu tượng chung cho mọi loại filesystem
        ↓
Filesystem cụ thể (ext4, xfs...) — dịch "file, offset" → "block nào"
        ↓
Block Layer (blk-mq) — kernel hiện đại dùng multi-queue: nhiều hàng đợi song song
        (đây là điểm khác NVMe tận dụng được rất tốt vì NVMe hỗ trợ nhiều queue thật,
         còn SATA/AHCI dù qua blk-mq vẫn bị giới hạn bởi bản thân giao thức chỉ có 1 queue)
        ↓
I/O Scheduler — sắp xếp lại thứ tự các yêu cầu (ví dụ gộp các block gần nhau
        để giảm số lần seek trên HDD; SSD/NVMe thường dùng scheduler "none"
        vì không có seek để tối ưu)
        ↓
Driver cụ thể (ahci/nvme/iscsi_tcp...) — dịch thành lệnh đúng giao thức
        ↓
Phần cứng thực thi
```

#### 8. Phần cứng trả kết quả về — ngắt (interrupt) vs polling

Đây là mảnh còn thiếu quan trọng: sau khi driver gửi lệnh xuống, CPU **không ngồi chờ** — nó làm việc khác, và ổ đĩa sẽ **báo ngắt (interrupt)** khi xong việc, CPU mới dừng lại xử lý kết quả. Đây là lý do I/O luôn là hoạt động **bất đồng bộ** ở tầng thấp nhất. (Bạn đã gặp khái niệm liên quan — "polling" của SPDK là cách **khước từ** cơ chế ngắt này, chủ động ngồi hỏi liên tục thay vì chờ ngắt, đổi lấy latency thấp hơn bằng cách đốt CPU liên tục.)