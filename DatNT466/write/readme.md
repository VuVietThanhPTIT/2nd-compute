1. Luồng Host-write:Ứng dụng trực tiếp trên Host ghi xuống ổ đĩa (O_DIRECT, qua iSCSI/multipath, có thể test cả iscsi-blk lẫn iscsi-tcp).Mục đích: Đo và hiểu chi phí cơ sở (Baseline) của bản thân hệ thống lưu trữ iSCSI khi chưa có ảo hóa.
2. Luồng VM-write:Ứng dụng trong Guest VM ghi xuống ổ ảo (write, O_DIRECT, cache=none).Dữ liệu qua Guest Kernel  virtio-blk  KVM/QEMU Host (cache=none, io=native)  backend iSCSI/multipath  Target.Mục đích: Chỉ ra ranh giới ảo hóa đã làm phình to overhead và làm trễ I/O ở những bước nào.

## Kết luận:

Nút thắt cổ chai khiến iSCSI truyền thống chậm hơn NVMe/SPDK và khiến VM-write chậm hơn Host-write KHÔNG PHẢI do CPU tốn công memcpy payload. Bản chất payload đều được truyền bằng con trỏ và kéo bằng DMA. Điểm nghẽn thực sự nằm ở:

+ hi phí xử lý Metadata: Quá nhiều tầng bọc/mở gói (CDB, PDU, TCP, IP).

+ Tranh chấp khóa (Lock Contention): Các CPU Core tranh nhau các hàng đợi của Kernel.

+ Chuyển đổi ngữ cảnh & Ngắt: VM-Exit, Syscall, SoftIRQ, và MSI-X Interrupt làm vỡ vụn bộ nhớ đệm CPU L1/L2/L3 (Cache Pollution).


Đếm chính xác: Có mấy lần memcpy payload?
Khi làm báo cáo kỹ thuật, bạn cần phân biệt rạch ròi:

+ Payload memcpy: Sao chép nội dung dữ liệu thật (4096 bytes dữ liệu).

+ Metadata copy: Sao chép các cấu trúc lệnh (mô tả vài chục bytes như CDB, SQE, Headers).
Mentor bảo bạn "đếm `memcpy`" không hề sai, và đây thực chất là một **bài test tư duy phản biện kinh điển** trong kỹ thuật hệ thống: kéo bạn từ "lý thuyết trên giấy" xuống "thực tế vận hành của phần cứng".

Về mặt lý thuyết, `O_DIRECT` hứa hẹn Zero-Copy payload. Nhưng trên hệ thống chạy thật, việc bắt bạn đi săn từng lệnh `memcpy` nhằm mục đích bóc trần 4 sự thật kỹ thuật sau:

### 1. Vạch trần các cú "copy ngầm" (Hidden Copies) mà lý thuyết thường giấu

Trên slide kiến trúc, mũi tên nối từ User Space xuống Kernel rồi ra NIC luôn được vẽ là Zero-Copy. Nhưng trong thực tế, chỉ cần một sai lệch nhỏ về cấu hình, Kernel và QEMU sẽ âm thầm kích hoạt `memcpy` để cứu hệ thống khỏi crash:

* **Bounce Buffer trong Linux Kernel (SWIOTLB):**
Nếu bộ đệm của ứng dụng không được căn chỉnh (unaligned) chuẩn 512B/4KiB, hoặc nếu phần cứng mạng/storage chỉ hỗ trợ DMA 32-bit trong khi RAM của bạn nằm ở dải 64-bit cao, Kernel không thể đưa địa chỉ đó cho phần cứng. Kernel buộc phải cấp phát một vùng đệm trung gian (**Bounce Buffer**) và gọi `memcpy` toàn bộ 4 KiB payload sang đó trước khi DMA.
* **QEMU IOVector Flattening (Tại ranh giới VM):**
Trong máy ảo, ứng dụng phát I/O có thể đưa vào một danh sách các vùng nhớ phân mảnh (Scatter-Gather). Khi sang QEMU, nếu backend không hỗ trợ ánh xạ phân tán trực tiếp từ Guest Physical Address sang Host Virtual Address, QEMU sẽ gọi hàm `qemu_iovec_to_buf()` để **`memcpy` gom toàn bộ các mảnh dữ liệu rải rác thành một khối liền mạch** trước khi gọi syscall xuống Host.
* **Sự bất đối xứng giữa Chiều Ghi (TX) và Chiều Đọc (RX):**
Chiều ghi (Write) với `O_DIRECT` rất dễ đạt Zero-Copy vì ứng dụng nắm quyền chuẩn bị RAM trước. Nhưng **chiều đọc (Read)** qua iSCSI TCP truyền thống thì khác: gói tin mạng từ Target bay về NIC được đổ vào các `sk_buff` do driver mạng cấp phát. Để đưa dữ liệu từ `sk_buff` vào buffer của ứng dụng, Kernel gần như bắt buộc phải chạy một lệnh **`copy_to_user` (chính là `memcpy`)**.

---

### 2. Buộc bạn phải lập "Bản đồ đối chiếu địa chỉ bộ nhớ"

Cách duy nhất để bạn chứng minh một đoạn đường có `memcpy` hay không là **soi địa chỉ RAM (Pointer Tracing)**:

* Bạn lấy địa chỉ `Host PA` của trang RAM lúc `pin_user_pages()`.
* Bạn soi tiếp địa chỉ nạp vào `struct bio`.
* Bạn soi tiếp địa chỉ nạp vào `SG-List` của SCSI.
* Bạn soi tiếp địa chỉ DMA nạp vào thanh ghi của Card NIC.

Nếu địa chỉ RAM vật lý ở tất cả các trạm trên **hoàn toàn trùng khớp một con số**, bạn có bằng chứng thép khẳng định: *"Đoạn này đạt Zero-Copy tuyệt đối, chỉ truyền con trỏ"*. Nếu ở trạm nào con số địa chỉ bị đổi, bạn biết ngay tại đó đã phát sinh một lần `memcpy`. Đây là kỹ năng phân tích chuyên sâu mà một kỹ sư Storage/Cloud bắt buộc phải có.

---

### 3. Nhận diện "Cơn bão Metadata Copy" làm ô nhiễm CPU Cache

Mentor dùng từ "đếm `memcpy`" theo nghĩa rộng của việc **sao chép bộ nhớ**:

* Một I/O 4 KiB payload có thể không bị copy nội dung `AAAA...`.
* Nhưng để đưa được 4 KiB đó đi, CPU phải liên tục cấp phát, khởi tạo và copy hàng loạt khối metadata:

$$\text{Virtq Desc} \longrightarrow \text{QEMU iocb} \longrightarrow \text{struct bio} \longrightarrow \text{struct request} \longrightarrow \text{SCSI CDB} \longrightarrow \text{iSCSI PDU} \longrightarrow \text{TCP sk\_buff}$$


* Mỗi lần tạo struct và copy header vài chục byte là một lần CPU đọc/ghi vào bộ nhớ đệm L1/L2. Khi tải cao chạm ngưỡng 500.000 đến 1.000.000 IOPS, **tổng lượng CPU tiêu tốn cho việc sao chép metadata này lớn tương đương hoặc vượt qua cả việc copy payload**, gây ra hiện tượng CPU Cache Thrashing (tràn và xóa đè bộ nhớ đệm vi xử lý).

---

### 4. Tạo "Bàn đạp đối chiếu" hoàn hảo để nâng tầm giải pháp SPDK

Nếu bạn không đo đạc và chỉ ra được các điểm `memcpy` (dù là payload ngầm hay metadata) trong baseline cũ, bạn sẽ không giải thích được vì sao SPDK lại nhanh hơn.

Khi đưa vào báo cáo, cấu trúc lập luận của bạn sẽ cực kỳ chặt chẽ:

1. **Thực trạng Baseline cũ (QEMU + Kernel iSCSI):**
*"Qua đo đạc thực nghiệm bằng `perf`, hệ thống baseline tiêu tốn X% CPU cho các hàm sao chép bộ nhớ (`memcpy_erms`, `copy_user_enhanced_fast_string`) do cơ chế quản lý buffer của QEMU và đóng gói đa tầng iSCSI/TCP."*
2. **Giải pháp SPDK:**
*"SPDK triệt tiêu 100% các điểm copy này nhờ:*
* *`Shared Hugepages`: QEMU và SPDK dùng chung một vùng RAM vật lý cố định, SPDK đọc/ghi thẳng vào bộ nhớ máy ảo mà không qua bounce buffer.*
* *`DPDK Mempool`: Bộ nhớ cho metadata (`spdk_bdev_io`) được cấp phát cố định sẵn theo từng CPU Core, tái sử dụng liên tục mà không tạo mới.*
* *`RDMA Zero-Copy`: Bỏ qua hoàn toàn tầng socket mạng, card Mellanox bốc thẳng từ RAM máy ảo đẩy sang RAM NetApp."*



---

### Cách bạn thực hành đo kiểm ngay trong bài Lab

Khi chạy `fio` trên máy test, bạn không cần phải đoán mò. Hãy bật công cụ `perf` của Linux lên để phần mềm tự "khai báo" xem CPU có đang tốn thời gian chạy `memcpy` hay không:

```bash
# Soi trực tiếp các hàm đang ngốn CPU nhiều nhất trong lúc fio đang chạy
perf top

# Hoặc ghi lại call-graph để xem ai là kẻ gọi memcpy
perf record -g -a sleep 10
perf report --stdio | grep -E "memcpy|copy_user"

```

Nếu trong kết quả xuất hiện các hàm như `memcpy_erms`, `copy_user_generic_string`, hoặc `qemu_iovec_to_buf`, bạn chỉ cần chụp màn hình lại và đính kèm vào báo cáo. Đó chính là **bằng chứng thực nghiệm xác đáng nhất** trả lời cho câu hỏi của mentor.

