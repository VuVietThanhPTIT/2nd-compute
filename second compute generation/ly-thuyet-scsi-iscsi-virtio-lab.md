# Tài liệu lý thuyết: SCSI/iSCSI, virtio-blk và Lab iSCSI-KVM-fio

Tài liệu tổng hợp toàn bộ thuật ngữ đã dùng trong lab baseline (storage host qua iSCSI → KVM host với virtio-blk → benchmark fio), giải thích từng khái niệm theo: **là gì / đặc điểm-tính chất / để làm gì / hoạt động như thế nào**.

---

# PHẦN 1 — HỌ SCSI VÀ iSCSI

## 1.1. SCSI (Small Computer System Interface)

**Là gì:** Một chuẩn công nghiệp định nghĩa **tập lệnh (command set)** để máy tính giao tiếp với thiết bị lưu trữ và các thiết bị ngoại vi khác (đĩa, băng từ, máy quét...), ra đời từ đầu thập niên 1980.

**Đặc điểm/tính chất:**
- Là tập lệnh **trừu tượng hoá** — không quan tâm dữ liệu truyền qua dây gì, chỉ định nghĩa "ý nghĩa của lệnh" (ví dụ READ(10), WRITE(10), INQUIRY, REPORT LUNS, TEST UNIT READY).
- Hỗ trợ **command queueing** — có thể xếp hàng nhiều lệnh cùng lúc, không bắt buộc chờ lệnh trước xong mới gửi lệnh sau (khác ATA/IDE nguyên thuỷ vốn đơn giản hơn).
- Hỗ trợ **nhiều loại thiết bị khác nhau** trên cùng 1 bus/hệ thống định danh (đĩa, băng từ, CD-ROM, scanner) — không chỉ riêng ổ đĩa.
- Có mô hình phân cấp: **Initiator** (bên khởi tạo yêu cầu, thường là máy chủ) và **Target** (bên nhận yêu cầu và trả lời, thường là thiết bị lưu trữ).

**Để làm gì:** Cung cấp 1 "ngôn ngữ chung" giữa hệ điều hành và thiết bị lưu trữ, để hệ điều hành không cần viết code riêng cho từng hãng sản xuất ổ đĩa — chỉ cần driver hiểu đúng tập lệnh SCSI là giao tiếp được với mọi thiết bị tuân theo chuẩn này.

**Hoạt động như thế nào:** Initiator đóng gói 1 lệnh SCSI (gọi là CDB — Command Descriptor Block, có cấu trúc byte cố định) gửi xuống target qua lớp vật lý (bus song song thời kỳ đầu, sau này là SAS, Fibre Channel, hay iSCSI). Target giải mã CDB, thực thi (đọc/ghi dữ liệu thật), rồi trả về trạng thái hoàn thành (status) và dữ liệu (nếu là lệnh đọc).

---

## 1.2. LUN (Logical Unit Number)

**Là gì:** Một số định danh cho **1 đơn vị lưu trữ logic** nằm bên trong 1 SCSI target — tức là 1 target vật lý có thể "chứa" nhiều LUN khác nhau, mỗi LUN được hệ điều hành ở phía initiator nhìn thấy như 1 ổ đĩa độc lập riêng biệt.

**Đặc điểm/tính chất:**
- LUN là khái niệm **logic**, không nhất thiết ứng với 1 ổ đĩa vật lý cụ thể — 1 hệ thống SAN có thể tạo ra hàng chục LUN từ 1 pool đĩa vật lý lớn, mỗi LUN thực chất chỉ là 1 vùng được cấp phát.
- Đánh số bắt đầu từ 0 (LUN 0 thường bắt buộc phải tồn tại, dùng để trả lời các lệnh quản trị chung của target).

**Để làm gì:** Cho phép 1 target export ra **nhiều "ổ đĩa ảo" độc lập** cho các máy chủ khác nhau dùng, mà không cần mỗi máy chủ phải là 1 target riêng biệt — tối ưu hoá việc chia sẻ hạ tầng lưu trữ.

**Liên quan tới lab:** Trong targetcli, bước `luns create /backstores/block/disk1` chính là tạo ra 1 LUN (mặc định LUN 0) trỏ tới backstore đã tạo — đây là bước khiến target thực sự "có dữ liệu để export", trước đó target chỉ là 1 cái tên (IQN) rỗng.

---

## 1.3. iSCSI (Internet SCSI)

**Là gì:** Giao thức đóng gói (encapsulate) các lệnh SCSI để truyền đi qua mạng TCP/IP thông thường, chuẩn hoá năm 2003 (RFC 3720).

**Đặc điểm/tính chất:**
- Không phát minh lại tập lệnh lưu trữ — dùng nguyên tập lệnh SCSI, chỉ thay đổi **lớp vận chuyển** (transport layer) từ bus vật lý cục bộ sang mạng IP.
- Kế thừa mọi đặc tính tin cậy của TCP: đảm bảo gói tin tới đúng thứ tự, tự động phát hiện và truyền lại gói mất.
- Chạy trên cổng TCP tiêu chuẩn 3260.
- Rẻ hơn Fibre Channel (giao thức SCSI-qua-mạng ra đời trước, dùng cáp quang chuyên dụng) vì tận dụng hạ tầng Ethernet có sẵn.

**Để làm gì:** Cho phép 1 máy chủ truy cập ổ đĩa/LUN nằm ở 1 máy khác (thậm chí ở xa qua WAN) **như thể** ổ đĩa đó cắm trực tiếp vào máy mình — ứng dụng/OS phía trên hoàn toàn không phân biệt được đó là ổ local hay ổ iSCSI.

**Hoạt động như thế nào:** Initiator đóng gói CDB SCSI vào 1 cấu trúc gọi là **PDU (Protocol Data Unit)** của iSCSI, gửi qua TCP connection tới target. Target giải mã PDU, lấy ra CDB SCSI bên trong, xử lý y hệt SCSI thông thường, rồi đóng gói kết quả ngược lại thành PDU trả về.

---

## 1.4. IQN (iSCSI Qualified Name)

**Là gì:** Chuỗi định danh **duy nhất toàn cầu** cho 1 iSCSI node (có thể là target hoặc initiator), theo cú pháp chuẩn: `iqn.<năm-tháng>.<domain đảo ngược>:<định danh tuỳ chọn>`.

Ví dụ: `iqn.2026-09.local.fptcloud:storage.target1`

**Đặc điểm/tính chất:**
- Phần `<năm-tháng>` thường là thời điểm tổ chức sở hữu domain đó đăng ký domain (không bắt buộc chính xác tuyệt đối trong lab nội bộ).
- Phần `<domain đảo ngược>` dùng domain thật của tổ chức viết ngược (giống package Java) để đảm bảo không trùng với tổ chức khác trên toàn cầu.
- Không gắn với địa chỉ IP — 1 IQN có thể "di chuyển" qua nhiều IP khác nhau theo thời gian (đây là điểm khác biệt quan trọng so với việc định danh bằng IP).

**Để làm gì:** Đóng vai trò "tên" để 2 bên iSCSI (initiator/target) nhận diện lẫn nhau ở tầng logic, độc lập với việc hạ tầng mạng phía dưới thay đổi ra sao (đổi IP, đổi VLAN...).

**Liên quan tới lab:** IQN của target (khai trong `iscsi create <iqn>`) và IQN của initiator (khai trong `/etc/iscsi/initiatorname.iscsi`) là 2 chuỗi bắt buộc phải khớp đúng với ACL đã cấu hình — sai 1 ký tự khiến login thất bại.

---

## 1.5. TPG (Target Portal Group)

**Là gì:** Một nhóm cấu hình gắn với 1 target, gồm: danh sách portal (IP:port) mà target lắng nghe, danh sách LUN được export qua nhóm này, và danh sách ACL kiểm soát truy cập.

**Đặc điểm/tính chất:**
- Mỗi target luôn có ít nhất 1 TPG (mặc định là `tpg1`).
- 1 target có thể có nhiều TPG để phục vụ các cấu hình mạng/policy truy cập khác nhau cho cùng 1 target — nhưng với lab đơn giản, chỉ cần 1 TPG.

**Để làm gì:** Tách biệt "target là ai" (định danh bằng IQN) khỏi "target lắng nghe ở đâu, ai được truy cập" (cấu hình trong TPG) — cho phép linh hoạt thay đổi network/access policy mà không cần đổi tên target.

---

## 1.6. Portal

**Là gì:** Tổ hợp địa chỉ IP + port mà target lắng nghe kết nối iSCSI đến — trong lab là `<storage_ip>:3260`.

**Đặc điểm/tính chất:** 1 target có thể có nhiều portal (nhiều IP khác nhau, ví dụ khi có nhiều NIC) — initiator có thể chọn kết nối qua bất kỳ portal nào để tới cùng 1 target, hỗ trợ multipath.

**Để làm gì:** Là "cổng vào" vật lý/mạng để initiator liên hệ được với target — không có portal thì dù target đã cấu hình đầy đủ LUN/ACL, không ai kết nối tới được.

---

## 1.7. Backstore

**Là gì:** Trong LIO, backstore là lớp ánh xạ giữa **1 nguồn dữ liệu thật** (block device, file, hay vùng RAM) và **1 "object"** mà LIO có thể gắn vào LUN để export ra ngoài.

**Đặc điểm/tính chất — các loại backstore:**
- `block`: trỏ thẳng tới 1 block device (LV, partition, hay ổ nguyên) — overhead thấp nhất, dùng cho lab benchmark chính.
- `fileio`: trỏ tới 1 file nằm trên filesystem của storage host — có thêm buffering của filesystem host, phù hợp khi không có sẵn raw device.
- `ramdisk`: cấp phát 1 vùng RAM làm backstore — cực nhanh, dùng để đo "trần" hiệu năng thuần của giao thức (như đã làm ở Phần 0 network baseline).

**Để làm gì:** Tách biệt "dữ liệu lưu ở đâu" (backstore) khỏi "dữ liệu được export ra ngoài như thế nào" (target/LUN) — cho phép đổi loại lưu trữ bên dưới mà không ảnh hưởng tới cấu hình iSCSI phía trên.

---

## 1.8. ACL (Access Control List, trong ngữ cảnh iSCSI)

**Là gì:** Danh sách các IQN của initiator được phép truy cập vào 1 TPG cụ thể.

**Đặc điểm/tính chất:** Mặc định LIO hoạt động theo mô hình **whitelist** — initiator không có trong ACL sẽ bị từ chối login dù đã discovery thành công (thấy được target tồn tại, nhưng không login được).

**Để làm gì:** Kiểm soát ai được phép mount LUN nào — vì bản thân network layer (ai route được tới IP đó) không đủ để giới hạn quyền truy cập, cần thêm 1 lớp kiểm soát riêng ở tầng logic iSCSI.

---

## 1.9. Initiator và iscsid/iscsiadm

**Initiator là gì:** Phần mềm phía client của iSCSI, chạy trên máy muốn "mượn" ổ đĩa từ xa.

**iscsid:** Daemon chạy nền, chịu trách nhiệm **duy trì session** (giữ TCP connection sống, tự động reconnect khi mất kết nối, gửi keep-alive).

**iscsiadm:** Công cụ dòng lệnh để **điều khiển** iscsid — ra lệnh discovery, login, logout, xem trạng thái session. Bản thân `iscsiadm` không giữ session, nó chỉ gửi yêu cầu cho `iscsid` xử lý.

**Vì sao tách 2 thành phần:** Tách biệt "control plane" (ra lệnh, chạy 1 lần rồi thoát) khỏi "data plane" (chạy liên tục, giữ trạng thái) — nguyên tắc thiết kế phổ biến trong hệ thống mạng/phân tán.

**Discovery vs Login — 2 bước khác nhau:**
- **Discovery** (`sendtargets`): hỏi 1 portal "mày export IQN nào" — trả lời bằng danh sách IQN có thể có nhiều target.
- **Login**: chọn đúng 1 IQN cụ thể, thiết lập phiên làm việc thật (session) để bắt đầu trao đổi lệnh SCSI.

---

# PHẦN 2 — VIRTIO VÀ VIRTIO-BLK

## 2.1. Ảo hoá (Virtualization) — bối cảnh chung

**Là gì:** Kỹ thuật cho phép 1 phần cứng vật lý (CPU, RAM, thiết bị I/O) phục vụ nhiều "máy ảo" (VM) tưởng rằng mỗi VM sở hữu toàn bộ phần cứng riêng.

**KVM (Kernel-based Virtual Machine):** Module trong kernel Linux, tận dụng phần cứng hỗ trợ ảo hoá của CPU (Intel VT-x/AMD-V) để bẫy (trap) các lệnh nhạy cảm của guest và chuyển quyền xử lý cho hypervisor — đây là cơ chế **VM Exit**.

**QEMU:** Chương trình chạy ở userspace, emulate các thiết bị (disk, network, USB...) cho VM, phối hợp với KVM để VM có đầy đủ "phần cứng ảo" hoạt động được.

## 2.2. VM Exit

**Là gì:** Sự kiện CPU tự động chuyển quyền điều khiển từ code đang chạy trong guest ra ngoài cho hypervisor, xảy ra khi guest thực thi 1 lệnh "nhạy cảm" (ví dụ truy cập thiết bị I/O được emulate).

**Đặc điểm/tính chất:** Mỗi lần VM Exit tốn chi phí đáng kể (lưu/khôi phục trạng thái CPU, chuyển ngữ cảnh) — hàng nghìn cycle CPU cho 1 lần trap, dù bản thân công việc xử lý sau đó có thể rất nhỏ.

**Để làm gì:** Là cơ chế bắt buộc để hypervisor "kiểm soát" được guest — không có VM Exit, guest có thể truy cập trực tiếp phần cứng thật, phá vỡ cách ly giữa các VM.

**Vấn đề gây ra:** Nếu 1 thiết bị ảo (như disk emulate kiểu IDE/SATA) yêu cầu VM Exit cho từng thao tác nhỏ (ghi từng thanh ghi), tần suất VM Exit rất cao với workload I/O dày đặc — đây chính là động lực ra đời của virtio.

## 2.3. Paravirtualization và Virtio

**Paravirtualization là gì:** Thay vì giả lập 1 thiết bị phần cứng "có thật" (emulation đầy đủ), thiết kế 1 giao diện thiết bị **biết trước là đang chạy trong môi trường ảo hoá**, tối ưu riêng cho việc guest-host giao tiếp hiệu quả, thay vì bắt chước hành vi phần cứng thật.

**Virtio là gì:** Chuẩn (specification) mở cho paravirtualized device, ban đầu do Rusty Russell phát triển, nay được chuẩn hoá bởi 1 uỷ ban kỹ thuật riêng — bao gồm virtio-blk (đĩa), virtio-net (mạng), virtio-scsi, virtio-gpu...

**Đặc điểm/tính chất:**
- Guest cần cài **driver virtio riêng** (khác driver emulate phần cứng thật) — hầu hết kernel Linux hiện đại có sẵn.
- Giao tiếp qua **vùng bộ nhớ chia sẻ** (shared memory) giữa guest và backend (QEMU), không phải thông qua giả lập từng thanh ghi phần cứng.

**Để làm gì:** Giảm số lần VM Exit và tổng chi phí ảo hoá I/O, bằng cách cho phép guest gửi **nhiều yêu cầu I/O gộp lại** trước khi cần 1 lần "báo hiệu" (kick) duy nhất, thay vì mỗi thao tác nhỏ đều gây ra 1 lần trap riêng.

## 2.4. Virtqueue

**Là gì:** Cấu trúc dữ liệu trung tâm của virtio — về bản chất là 1 **ring buffer** (vùng nhớ dạng vòng) nằm trong bộ nhớ chia sẻ giữa guest và backend, dùng để guest đặt các yêu cầu I/O vào và nhận kết quả trả về.

**Đặc điểm/tính chất:**
- Gồm nhiều phần: bảng mô tả (descriptor table — trỏ tới vùng bộ nhớ chứa dữ liệu thật), **avail ring** (nơi guest ghi "tôi vừa đặt thêm 1 request"), **used ring** (nơi backend ghi "tôi đã xử lý xong request nào").
- Không cần khoá (lock) giữa guest và backend nhờ thiết kế theo mô hình producer-consumer với các chỉ số (index) riêng biệt cho từng bên ghi.
- 1 thiết bị virtio có thể có nhiều virtqueue (ví dụ virtio-blk có thể có nhiều queue tương ứng số vCPU, giống mô hình multi-queue của blk-mq).

**Để làm gì:** Là "kênh giao tiếp" hiệu quả để guest gửi nhiều I/O request liên tiếp mà backend có thể xử lý bất đồng bộ, không cần đồng bộ hoá phức tạp giữa 2 bên.

**Hoạt động như thế nào:**
```
1. Guest driver (virtio-blk) chuẩn bị dữ liệu, ghi descriptor mô tả request vào virtqueue
2. Guest cập nhật avail ring, báo "có request mới"
3. Guest gửi 1 tín hiệu "kick" (qua ioeventfd) để đánh thức backend — ĐÂY LÀ LẦN VM EXIT DUY NHẤT
   cho dù có nhiều request đã được đặt vào hàng đợi trước đó
4. QEMU (backend) đọc virtqueue từ vùng nhớ chia sẻ, lấy ra request, thực thi I/O thật
5. QEMU ghi kết quả vào used ring, gửi ngắt (irqfd) báo guest "đã xong"
6. Guest driver đọc used ring, biết request nào hoàn thành, trả kết quả lên tầng trên
```

## 2.5. virtio-blk cụ thể

**Là gì:** Thiết bị đĩa paravirtualized theo chuẩn virtio, cung cấp giao diện block device cho guest.

**Đặc điểm/tính chất:**
- Guest thấy thiết bị dưới dạng `/dev/vdX` (khác `/dev/sdX` của thiết bị SCSI/SATA emulate).
- Driver `virtio_blk` trong kernel guest nói chuyện qua virtqueue thay vì giả lập thanh ghi IDE/AHCI.
- Hỗ trợ nhiều tham số cấu hình phía QEMU ảnh hưởng trực tiếp hiệu năng: cache mode, io mode (đã nói ở phần dưới).

**Để làm gì:** Cung cấp đường truyền I/O đĩa hiệu quả nhất có thể trong mô hình QEMU/KVM truyền thống (chưa tính tới vhost-user-blk sẽ học ở lab sau, vốn bỏ luôn QEMU khỏi vòng lặp xử lý mỗi request).

**So sánh với virtio-scsi:** virtio-blk đơn giản hơn (1 thiết bị = 1 disk), còn virtio-scsi mô phỏng cả tầng SCSI đầy đủ hơn (hỗ trợ nhiều LUN qua 1 controller, gần giống cách bạn đã học ở phần SCSI) — lab này dùng virtio-blk vì đơn giản và đủ dùng cho benchmark cơ bản.

## 2.6. cache='none' và io='native' trong domain XML

**cache='none' — là gì và để làm gì:**
- Tắt hoàn toàn page cache của **host OS** cho riêng disk này (kỹ thuật dùng cờ `O_DIRECT` khi QEMU mở file/device backing).
- Mục đích: đảm bảo mọi I/O từ guest đều thực sự chạm xuống thiết bị vật lý (hoặc iSCSI target) phía dưới, không bị "trả lời giả" bởi cache RAM của host — cần thiết để benchmark đo đúng hiệu năng thật.

**io='native' — là gì và để làm gì:**
- Chọn cơ chế Linux AIO (Asynchronous I/O) thật để QEMU gửi yêu cầu I/O xuống host kernel — cho phép gửi nhiều request cùng lúc mà không cần 1 thread riêng cho mỗi request.
- So với `io='threads'` (dùng thread pool để giả lập bất đồng bộ): `native` phản ánh đúng khả năng song song hoá thật của toàn bộ datapath bên dưới; `threads` bị giới hạn giả tạo bởi số lượng thread trong pool.

---

# PHẦN 3 — LƯU TRỮ, LVM VÀ FIO (tóm tắt lại có hệ thống)

## 3.1. Block và Block Device

**Là gì:** Block là đơn vị dữ liệu kích thước cố định (512B/4096B) — đơn vị nhỏ nhất mà phần cứng lưu trữ (HDD/SSD) có thể đọc/ghi. Block device là loại thiết bị bắt buộc thao tác theo bội số của block size, hỗ trợ truy cập ngẫu nhiên.

**Vì sao tồn tại:** HDD bị ràng buộc bởi cơ chế cơ học (sector là đơn vị nhỏ nhất đầu từ đọc được trọn vẹn); SSD bị ràng buộc bởi cơ chế điện tử của NAND (page là đơn vị ghi, ứng với 1 wordline vật lý). Cả 2 hội tụ về cùng nguyên lý: không thể thao tác nhỏ hơn 1 cụm cố định.

## 3.2. LVM — PV/VG/LV/PE/LE

**PV (Physical Volume):** 1 block device thật đã được LVM "đóng dấu" metadata để nhận diện làm thành viên LVM.

**VG (Volume Group):** "Hồ chứa" logic gộp nhiều PV — bản chất chỉ là 1 bản ghi metadata, không phải device riêng.

**LV (Logical Volume):** Device thật do kernel tạo ra (qua device-mapper), ánh xạ từ không gian logic sang không gian vật lý của VG.

**PE (Physical Extent) / LE (Logical Extent):** Đơn vị cấp phát nhỏ nhất của LVM (mặc định 4MiB) — LV luôn được cấp phát theo số nguyên PE/LE, không thể lẻ. LE ánh xạ 1-1 tới PE thông qua bảng mapping lưu trong metadata VG.

**device-mapper:** Framework kernel thực thi việc "chặn và dịch lại" mọi I/O tới LV, viết lại địa chỉ theo bảng ánh xạ LE→PE, trước khi chuyển tiếp xuống driver thật.

## 3.3. Page cache và O_DIRECT

**Page cache:** Vùng RAM kernel dùng để giữ bản sao dữ liệu đọc/ghi gần đây, tăng tốc truy cập lặp lại.

**O_DIRECT:** Cờ mở file/device yêu cầu bypass page cache — dữ liệu đi thẳng giữa buffer ứng dụng và thiết bị, không qua trung gian RAM cache. Cần thiết khi benchmark để đo đúng hiệu năng thiết bị thật, không bị "ảo" bởi cache.

## 3.4. AIO (Asynchronous I/O) và Linux libaio

**Vấn đề của I/O đồng bộ:** Gọi `read()`/`write()` truyền thống là **blocking** — thread gọi phải dừng chờ tới khi I/O xong. Muốn N I/O song song cần N thread.

**AIO giải quyết:** Cho phép **submit** nhiều I/O request liên tiếp mà không chờ, sau đó **poll hoặc nhận thông báo** khi từng request hoàn thành — tách "gửi yêu cầu" khỏi "chờ kết quả", đạt được queue depth cao mà không cần tương ứng nhiều thread.

## 3.5. fio và các khái niệm benchmark

**IOPS (I/O Operations Per Second):** Số lượng thao tác I/O hoàn thành mỗi giây — chỉ số quan trọng nhất với workload block nhỏ, ngẫu nhiên (ví dụ database).

**Throughput/Bandwidth (MB/s):** Lượng dữ liệu truyền được mỗi giây — chỉ số quan trọng nhất với workload block lớn, tuần tự (ví dụ streaming, backup).

**iodepth (Queue Depth):** Số lượng I/O request "đang bay" (đã gửi, chưa nhận kết quả) cùng lúc — tăng iodepth giúp tận dụng "độ trễ ẩn" (latency hiding), tăng IOPS tới khi phần cứng bão hoà.

**clat (Completion Latency):** Thời gian từ lúc I/O được submit tới lúc hoàn thành — chỉ số latency quan trọng nhất, phản ánh đúng độ trễ thực của toàn bộ datapath.

**Percentile latency (P50/P99/P99.99):** Thay vì chỉ nhìn trung bình (dễ bị "lu mờ" bởi số đông), percentile cho biết "trong 100 request, request tệ thứ N mất bao lâu" — quan trọng vì hệ thống production quan tâm tới trải nghiệm tệ nhất (tail latency), không chỉ trường hợp trung bình.

---

# PHẦN 4 — LÝ THUYẾT TỔNG THỂ CỦA LAB

## 4.1. Mục tiêu của lab

Dựng 1 datapath hoàn chỉnh theo đúng mô hình SAN truyền thống (dựa trên SCSI/iSCSI) — từ ổ đĩa vật lý trên storage host, qua LVM, qua LIO/iSCSI, qua mạng, tới KVM host, qua virtio-blk, vào VM — rồi đo hiệu năng bằng fio để có **số liệu baseline**, dùng làm mốc so sánh khi chuyển sang các datapath tối ưu hơn (nvme-tcp, RDMA, SPDK/vhost-user-blk) ở các lab tiếp theo trong project lớn của team.

## 4.2. Vì sao thứ tự các bước lại như vậy

**Network baseline (Phần 0) trước iSCSI:** Nếu không tách bạch, khi số liệu fio cuối cùng thấp, không thể biết nguyên nhân là network yếu, giao thức iSCSI có overhead cao, hay bản thân ổ đĩa chậm. Đo network thuần (ping, iperf3) và "trần iSCSI" (RAM disk backstore) trước giúp có 3 mốc để so sánh, cô lập rõ nguyên nhân khi phân tích số liệu cuối.

**Storage host trước KVM host:** Vì KVM host (initiator) cần có 1 target đã sẵn sàng để login vào — không thể test initiator khi chưa có gì ở đầu kia.

**cache=none/io=native trước khi benchmark:** Đây là bước "làm sạch biến nhiễu" — nếu không tắt cache, benchmark có thể đo trúng tốc độ RAM thay vì tốc độ datapath thật; nếu không dùng AIO thật, benchmark bị giới hạn giả tạo bởi mô hình thread.

**fio raw device, không format filesystem:** Tránh thêm 1 lớp overhead (metadata, journal của filesystem) không liên quan tới mục tiêu đo hiệu năng datapath thuần.

## 4.3. Sơ đồ luồng dữ liệu đầy đủ của lab

```
[fio trong Guest VM]
        │
[virtio_blk driver (guest kernel)] — ghi request vào virtqueue
        │
[QEMU block layer] — driver=qemu, cache=none (bypass page cache host), io=native (AIO thật)
        │
[Host kernel: iSCSI initiator (iscsid)] — đóng gói SCSI command thành iSCSI PDU
        │
[Network — TCP, cổng 3260]
        │
[LIO target (storage host)] — giải mã PDU, lấy SCSI command
        │  TPG → LUN → Backstore
[device-mapper (nếu backstore là LVM LV)] — dịch địa chỉ logic LE sang vật lý PE
        │
[Driver đĩa thật (ahci/nvme)] — thực thi lệnh vật lý
        │
[Ổ đĩa vật lý]
```

## 4.4. Từng lớp trong sơ đồ đóng góp overhead gì (liên hệ ngược lại phần 1-3)

| Lớp | Cơ chế | Overhead |
|---|---|---|
| virtqueue/virtio-blk | Gộp nhiều request, giảm VM Exit | Vẫn cần QEMU trung gian đọc/ghi virtqueue |
| iSCSI/TCP | Đóng gói SCSI command qua mạng | Độ trễ network + TCP overhead (ACK, retransmit khi mất gói) |
| LVM/device-mapper | Ánh xạ địa chỉ logic→vật lý | Thêm 1 bước tra bảng mapping |
| Page cache (nếu không tắt) | Cache RAM giúp truy cập lặp nhanh | Che giấu hiệu năng thật — phải tắt khi benchmark |

## 4.5. Ý nghĩa của việc lab này là baseline cho lộ trình SPDK/NVMe-oF

Mọi lớp trong bảng trên đều là mục tiêu mà các công nghệ ở lab sau tìm cách loại bỏ hoặc rút gọn:
- **vhost-user-blk** loại bỏ QEMU khỏi vòng lặp xử lý mỗi request (SPDK đọc thẳng virtqueue qua shared memory).
- **NVMe-oF/RDMA** thay thế TCP bằng RDMA (bypass kernel network stack, zero-copy), và thay tập lệnh SCSI cũ bằng NVMe (nhiều queue song song hơn).
- **SPDK (polling mode)** loại bỏ cơ chế ngắt (interrupt) của kernel, dùng vòng lặp polling liên tục để giảm latency.

Hiểu rõ từng khái niệm trong tài liệu này ở mức lý thuyết là điều kiện để hiểu **chính xác** từng công nghệ ở lab sau đang "cắt bớt" đúng lớp overhead nào trong sơ đồ trên, và vì sao việc cắt bớt đó lại giảm được latency/tăng IOPS.
