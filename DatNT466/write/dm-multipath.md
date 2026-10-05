**dm-multipath** (*Device Mapper Multipathing*) là một module của Linux Kernel cho phép gộp **nhiều đường truyền vật lý khác nhau** dẫn tới cùng một ổ đĩa logic thành **một thiết bị khối ảo duy nhất** (ví dụ: gom `/dev/sda` và `/dev/sdb` thành `/dev/mapper/mpath0`).

---

### 1. Bản chất: `dm-multipath` sinh ra để giải quyết vấn đề gì?

Trong môi trường lưu trữ mạng SAN (iSCSI, Fibre Channel):

* Tủ đĩa lưu trữ (Storage Target như NetApp, Dell EMC) tạo ra một ổ đĩa ảo gọi là **LUN** (*Logical Unit Number*).
* Để đảm bảo an toàn, máy chủ (Host) không chỉ cắm 1 sợi dây mạng tới tủ đĩa, mà cắm ít nhất 2 đường độc lập:
* **Đường 1:** Card mạng NIC 1 $\to$ Switch mạng 1 $\to$ Controller A của tủ đĩa.
* **Đường 2:** Card mạng NIC 2 $\to$ Switch mạng 2 $\to$ Controller B của tủ đĩa.



#### Vấn đề nếu KHÔNG có dm-multipath:

Linux phát hiện thiết bị theo chuẩn bus độc lập. Vì Host nhìn thấy cùng một LUN thông qua 2 cổng mạng khác nhau, Kernel sẽ ngây thơ nhận diện LUN đó thành **2 ổ đĩa riêng biệt**: `/dev/sda` và `/dev/sdb`.

* Nếu ứng dụng chia phân vùng hoặc ghi đồng thời vào cả `/dev/sda` và `/dev/sdb`, dữ liệu trên LUN sẽ bị **hỏng hoàn toàn (Data Corruption)** do tranh chấp ghi đè.
* Nếu ứng dụng chỉ dùng `/dev/sda` mà sợi cáp 1 bị đứt, I/O lập tức báo lỗi chết hệ thống dù đường thứ 2 (`/dev/sdb`) vẫn sống nguyên vẹn.

#### Vai trò của `dm-multipath`:

`dm-multipath` can thiệp vào tầng Block Layer để:

1. **Gộp thiết bị (Device Aggregation):** Ẩn `/dev/sda` và `/dev/sdb` đi, tạo ra một node ảo đại diện là `/dev/mapper/mpath0`. Ứng dụng và hệ điều hành chỉ thao tác trên `mpath0`.
2. **Dự phòng lỗi tức thì (Failover):** Đứt cáp đường 1 $\to$ tự động chuyển toàn bộ I/O sang đường 2 trong suốt với ứng dụng, không làm rớt tiến trình.
3. **Cân bằng tải (Load Balancing):** Phân phối luân phiên các request qua cả 2 card mạng để tận dụng gấp đôi băng thông.

---

### 2. Tại sao khi ghi trực tiếp Local Disk thì KHÔNG CẦN multipath?

Khi chuyển từ ghi mạng sang ghi ổ đĩa cắm trực tiếp trên bo mạch chủ (Local NVMe SSD), `dm-multipath` bị loại bỏ vì 3 lý do kỹ thuật cốt lõi:

#### A. Kiến trúc vật lý là Điểm-Điểm (Point-to-Point)

* Ổ NVMe cắm trực tiếp vào khe PCIe trên mainboard.
* Tuyến kết nối vật lý từ CPU tới chip điều khiển SSD chỉ có **duy nhất 1 đường độc đạo**:

$$\text{CPU Core} \longleftrightarrow \text{PCIe Root Complex} \longleftrightarrow \text{PCIe Lanes (x4)} \longleftrightarrow \text{NVMe Controller}$$


* Giữa CPU và SSD **không hề có 2 cổng mạng, 2 switch hay 2 card điều khiển** để lựa chọn hay chuyển đổi. Vì chỉ có 1 đường vật lý duy nhất, khái niệm "chọn đường" (*Path Selection*) hay "đa đường" (*Multipath*) hoàn toàn không có đối tượng để thực thi.

#### B. Không có sự trùng lặp thiết bị (No Device Ambiguity)

* Vì chỉ có một kết nối bus PCIe, Linux Kernel chỉ quét thấy **duy nhất một node thiết bị**: `/dev/nvme0n1`.
* Không có hiện tượng một ổ đĩa bị nhân bản thành nhiều file thiết bị khác nhau, nên Kernel không cần một tầng trung gian ảo (Device Mapper) đứng ra gộp hay bảo vệ dữ liệu.

#### C. Triệt tiêu chi phí phụ (Overhead) để tối ưu độ trễ nano-giây

Mỗi tầng phần mềm trong kernel đều phải trả giá bằng chu kỳ xung nhịp CPU:

* Dùng `dm-multipath` bắt buộc kernel phải: cấp phát thêm metadata `struct dm_rq_target_io`, chạy thuật toán tính tải của đường truyền, nhân bản request (`cloned request`), và móc hàm callback để bắt lỗi failover.
* Ổ cứng Local NVMe SSD được thiết kế để phục vụ I/O ở cấp độ micro-giây/nano-giây. Việc ép một ổ đĩa local chạy qua `dm-multipath` không mang lại bất kỳ lợi ích dự phòng nào, mà chỉ làm tốn chu kỳ CPU, phân mảnh bộ nhớ đệm CPU L1/L2/L3, và kéo tụt IOPS.

---

### Góc nhìn mở rộng: Local Disk có bao giờ dùng Multipath không?

Trong thực tế doanh nghiệp có một ngoại lệ: **Dual-port SAS hoặc Dual-port NVMe SSD**.

* Đây là các ổ đĩa chuyên dụng cho máy chủ cao cấp có 2 cổng kết nối vật lý độc lập trên cùng một thân ổ (chân cắm U.2 / U.3 hoặc EDSFF).
* Hai cổng này được cắm vào 2 bo mạch chủ khác nhau (mô hình 2 máy chủ chạy song song - High Availability Cluster) hoặc nối vào 2 HBA card khác nhau trên cùng một máy để chống cháy card điều khiển.
* Tuy nhiên, ngay cả với NVMe Dual-port, Linux hiện đại cũng ưu tiên dùng **NVMe Native Multipathing (ANA - Asymmetric Namespace Access)** tích hợp sẵn trong driver `nvme_core` thay vì dùng module `dm-multipath` truyền thống của tầng SCSI, nhằm giảm thiểu tối đa độ trễ.