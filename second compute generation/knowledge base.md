Mình giải thích theo đúng thứ tự dữ liệu đi qua, từ dưới lên, và ghi rõ chỗ nào trong lab của bạn tương ứng với phần lý thuyết nào.

## 0. Bức tranh tổng thể

```
fio (trong guest)
 └─ /dev/vdb                       driver virtio_blk trong guest
     └─ virtqueue                  vùng nhớ chia sẻ guest ⇄ QEMU
         └─ QEMU                   open("/dev/mapper/mpatha", O_DIRECT) + Linux AIO
             └─ dm-0 (mpatha)      device-mapper multipath
                 └─ /dev/sdb       SCSI disk, địa chỉ H:C:T:L
                     └─ iSCSI initiator (iscsid + iscsi_tcp)   1 session, 1 kết nối TCP
                         └─ TCP/IP  →  10.11.4.21:3260
                             └─ LIO target (iscsi_target_mod → target_core_file)
                                 └─ /storage/iscsi-backing.img → ext4 → sda
```

## 1. Block storage và SAN

- **Block storage** đưa cho client một "đĩa thô" đánh địa chỉ theo LBA (Logical Block Address). Client tự đặt filesystem hoặc database lên trên. Khác với file storage (NFS/SMB, server quản lý filesystem) và object storage (S3).
- **SAN** là mạng chuyên chở lệnh SCSI giữa server và storage. Fibre Channel chạy trên hạ tầng riêng, iSCSI giữ nguyên khái niệm nhưng chạy trên Ethernet/IP, nên rẻ hơn và dùng lại được mạng có sẵn.
- **SDS (Software-Defined Storage)**: dùng phần mềm (ở đây là LIO cộng file backing) để một server thường đóng vai storage array.

## 2. SCSI: nền tảng của iSCSI

- SCSI là mô hình client-server: **initiator** gửi lệnh tới một **LUN** của **target**. Lệnh gọi là **CDB** (Command Descriptor Block, 6 đến 16 byte, gồm opcode, LBA, độ dài). Target trả dữ liệu và status (`GOOD`, hoặc `CHECK CONDITION` kèm sense data).
- Các lệnh hay gặp: `INQUIRY` (hỏi vendor/product/định danh), `TEST UNIT READY`, `REPORT LUNS`, `READ CAPACITY(16)`, `READ(16)`/`WRITE(16)`, `SYNCHRONIZE CACHE` (flush).
- Kernel Linux xử lý mọi đĩa SCSI như nhau (SAS, SATA hay iSCSI) qua SCSI mid-layer và driver `sd`, chỉ khác lớp transport phía dưới. Vì vậy LUN iSCSI hiện ra `/dev/sdb` y như đĩa cắm trực tiếp.
- Địa chỉ **H:C:T:L** = Host adapter, Channel, Target, LUN. Mỗi session iSCSI tạo ra một "host" ảo. Ví dụ `33:0:0:0` ở lần đo trên mkolla00.
- **iSCSI = SCSI over TCP** (RFC 7143): nhét CDB vào các gói PDU rồi chạy trên TCP, cổng mặc định **3260**.

## 3. Định danh và mô hình đối tượng iSCSI

|Khái niệm|Trong lab|Ý nghĩa|
|---|---|---|
|Target|`iqn.2026-09.local.fptcloud:storage.target1`|"Máy chủ storage" logic. Một máy vật lý có thể chứa nhiều target.|
|Initiator|`iqn.2004-10.com.ubuntu:01:...` (file `/etc/iscsi/initiatorname.iscsi`)|Client.|
|Portal|`10.11.4.21:3260`|IP:port lắng nghe.|
|TPG|`tpg1`, tag `1` (kết quả discovery có dạng `10.11.4.21:3260,1`)|Nhóm gồm portal, LUN, ACL và thuộc tính. Số sau dấu phẩy là TPGT.|
|LUN|`lun0`|Đơn vị lưu trữ mà initiator thấy như một đĩa.|
|Session|1 phiên giữa 1 initiator và 1 target|Định danh bằng ISID (initiator cấp) và TSIH (target cấp).|
|Connection|1 kết nối TCP trong session|Một session có thể gồm nhiều connection (MC/S).|

**Cấu trúc IQN**: `iqn.YYYY-MM.<domain viết ngược>:<chuỗi tùy ý>`. `YYYY-MM` là tháng tổ chức đăng ký domain, `local.fptcloud` là domain đảo ngược, phần sau dấu `:` do bạn đặt. IQN chỉ là **tên**, không phải bí mật, ai biết IQN đều có thể giả mạo. Vì vậy môi trường thật cần CHAP và mạng storage tách riêng.

## 4. PDU: gói tin của iSCSI

```
+-----------------------------------------------+
| BHS  Basic Header Segment (48 byte)           |  opcode, flags, LUN, ITT, CmdSN, CDB...
| AHS  Additional Header Segment (tùy chọn)     |
| Header Digest (CRC32C, tùy chọn)              |
| Data Segment (dữ liệu / tham số / text)       |
| Data Digest   (CRC32C, tùy chọn)              |
+-----------------------------------------------+
```

|Initiator → Target|Opcode|Target → Initiator|Opcode|
|---|---|---|---|
|NOP-Out|0x00|NOP-In|0x20|
|SCSI Command|0x01|SCSI Response|0x21|
|Task Management|0x02|Task Mgmt Response|0x22|
|Login Request|0x03|Login Response|0x23|
|Text Request|0x04|Text Response|0x24|
|SCSI Data-Out|0x05|SCSI Data-In|0x25|
|Logout Request|0x06|Logout Response|0x26|
|||R2T (Ready To Transfer)|0x31|

Digest là tùy chọn (`None` hoặc `CRC32C`). Bật thì phát hiện được dữ liệu hỏng mà checksum TCP bỏ sót, đổi lại tốn CPU.

## 5. Vòng đời một session, gắn với lệnh bạn đã chạy

### 5.1. Discovery: `iscsiadm -m discovery -t sendtargets`

- open-iscsi mở một **discovery session** (SessionType=Discovery) tới portal, chưa cần biết IQN.
- Nó gửi **Text Request** `SendTargets=All`. Target trả **Text Response** chứa `TargetName=iqn...` và `TargetAddress=10.11.4.21:3260,1`. Rồi logout.
- Kết quả được ghi thành **node record** trong `/var/lib/iscsi/nodes/`. Đó là lý do lệnh `-o update -n node.startup` sửa được record, và `-o delete` xóa được nó.
- Discovery thường liệt kê target mà chưa kiểm tra ACL. ACL chỉ bị kiểm tra ở bước login.

### 5.2. Login: `iscsiadm ... --login`

Ba giai đoạn:

1. **Security negotiation**: chọn `AuthMethod`. Lab dùng `no-auth`. Nếu bật CHAP thì target gửi challenge, initiator trả `MD5(id + secret + challenge)`.
2. **Operational negotiation**: hai bên thương lượng các cặp `key=value` trong Data Segment.
3. **Full Feature Phase**: sau khi Login Response có status thành công, dòng `Login ... successful` mà bạn thấy chính là kết quả này.

Các tham số thương lượng quan trọng:

|Tham số|Ý nghĩa|
|---|---|
|`HeaderDigest` / `DataDigest`|Không kiểm tra hoặc CRC32C.|
|`MaxRecvDataSegmentLength`|Kích thước tối đa dữ liệu trong 1 PDU. I/O 1M sẽ bị chia thành nhiều PDU.|
|`InitialR2T` / `ImmediateData`|Ghi có cần chờ R2T không, có gửi kèm dữ liệu trong Command PDU không. Ảnh hưởng số vòng RTT của một lệnh ghi.|
|`FirstBurstLength` / `MaxBurstLength`|Lượng dữ liệu ghi gửi không cần xin phép và lượng tối đa mỗi lần R2T.|
|`MaxConnections`|Số kết nối TCP tối đa trong 1 session.|
|`ErrorRecoveryLevel`|ERL0 nghĩa là lỗi thì bỏ session và đăng nhập lại.|

**Kiểm tra ACL xảy ra ở đây.** Trong lab target ở chế độ `no-gen-acls`, nên initiator không có trong ACL sẽ bị Login Response từ chối với status nhóm lỗi initiator, ví dụ `0x0201` (authentication failure) hoặc `0x0202` (authorization failure). Đó chính là lỗi "initiator failed authorization" mà mình đã cảnh báo khi sai IQN.

### 5.3. Đọc và ghi (Full Feature Phase)

```
ĐỌC
Initiator                                        Target (LIO)
   | SCSI Command (READ(16), LBA, độ dài, ITT=x) ────────►|
   |◄──────────────── SCSI Data-In (một hoặc nhiều PDU) ───|
   |◄──────────────── SCSI Response (GOOD) ────────────────|   (thường gộp vào Data-In cuối)

GHI
   | SCSI Command (WRITE(16), ITT=y, kèm ImmediateData) ───►|
   |◄──────────────── R2T (offset, length ≤ MaxBurstLength) |   (nếu còn dữ liệu cần "xin phép")
   | SCSI Data-Out ────────────────────────────────────────►|
   |◄──────────────── SCSI Response (GOOD) ────────────────|
```

- Mỗi lệnh có **ITT** (Initiator Task Tag) riêng. Nhiều lệnh cùng bay song song và target có thể trả lời không theo thứ tự. Đây là nền tảng của **queue depth**.
- **CmdSN** tăng theo mỗi lệnh. Target công bố cửa sổ `[ExpCmdSN, MaxCmdSN]`, đó là **flow control ở mức lệnh**, giới hạn số lệnh đang bay trên một session. **StatSN** đánh số các response.
- Phía initiator, giới hạn nằm trong `/etc/iscsi/iscsid.conf`: `node.session.cmds_max` (thường mặc định 128) và `node.session.queue_depth` (thường mặc định 32 mỗi LUN). Kiểm tra bằng:

```bash
grep -E "queue_depth|cmds_max|replacement_timeout|noop_out" /etc/iscsi/iscsid.conf
cat /sys/block/sdb/device/queue_depth
```

### 5.4. Sau login, kernel quét LUN

Kernel gửi `REPORT LUNS`, `INQUIRY` (kể cả trang VPD 0x83), `READ CAPACITY(16)`, rồi driver `sd` tạo `/dev/sdb`.

- `INQUIRY` trả vendor `LIO-ORG` và product là **tên backstore** (`disk1`). Đó là dòng `LIO-ORG,disk1` trong `multipath -ll`.
- VPD 0x83 chứa định danh NAA, `scsi_id` in ra thành WWID `36001405b94ac0b4...`:
    - `3` do `scsi_id` thêm vào cho loại NAA.
    - `6` là NAA type 6 (IEEE Registered Extended).
    - `001405` là OUI của LIO.
    - Phần còn lại là serial (UUID sinh ra một lần khi tạo backstore).
- **WWID gắn với backstore, không gắn với target hay IP.** Vì vậy khi bạn đổi KVM host, LUN vẫn ra đúng cùng một WWID.

### 5.5. Keepalive, timeout và phục hồi

- Initiator định kỳ gửi **NOP-Out**, target trả **NOP-In**. Tham số `noop_out_interval` và `noop_out_timeout` (mặc định thường 5 giây mỗi cái). Không có phản hồi thì kết nối bị coi là hỏng.
- Khi đó session vào trạng thái phục hồi. Trong `replacement_timeout` (mặc định thường 120 giây), I/O tầng trên bị xếp hàng chờ. Hết thời gian mà chưa nối lại được, I/O mới trả lỗi.
- Trong lab, `multipath -ll` có `features='1 queue_if_no_path'`, nghĩa là dm-multipath **xếp hàng I/O vô hạn thay vì báo lỗi** khi mất hết đường. Hệ quả: mất mạng iSCSI thì VM không thấy lỗi mà **đứng hình** cho tới khi có lại đường. Đây là đánh đổi giữa "không mất dữ liệu" và "treo".
- Kết thúc bằng **Logout Request/Response**.

### 5.6. Nhiều initiator dùng chung một LUN

iSCSI cho phép nhiều session tới cùng LUN, nhưng LUN raw **không có cơ chế khóa ghi giữa các host**. Hai host cùng ghi sẽ hỏng dữ liệu, trừ khi dùng cluster filesystem hoặc SCSI-3 Persistent Reservation. Đó là lý do khi đổi KVM host mình yêu cầu logout host cũ và đổi ACL trước.

## 6. Phía target: LIO

**Kiến trúc**: `target_core_mod` (lõi: xử lý lệnh SCSI, LUN, ALUA, reservation) + `iscsi_target_mod` (phần iSCSI: login, PDU, session) + các module backstore (`target_core_iblock`, `target_core_file`, `target_core_rd_mcp`, `target_core_pscsi`). `targetcli` chỉ là công cụ điều khiển: nó ghi cấu hình vào **configfs** (`/sys/kernel/config/target/`). `saveconfig` lưu ra file JSON để service nạp lại sau reboot.

|Trong `targetcli`|Đối tượng LIO|Ý nghĩa|
|---|---|---|
|`/backstores/fileio/disk1`|Storage object|Nguồn dữ liệu: file `/storage/iscsi-backing.img`.|
|`/iscsi/iqn...`|iSCSI target|Định danh target.|
|`tpg1`|Target Portal Group|Chứa thuộc tính `no-gen-acls`, `no-auth`.|
|`portals/10.11.4.21:3260`|Network portal|Socket lắng nghe.|
|`luns/lun0`|LUN|Gắn storage object vào TPG.|
|`acls/<iqn>` → `mapped_lun0`|Node ACL + mapped LUN|Initiator nào được thấy LUN nào, quyền `rw`.|
|`alua/default_tg_pt_gp`|ALUA target port group|Trạng thái `Active/optimized`.|

- `no-gen-acls` (`generate_node_acls=0`): chỉ initiator có ACL mới vào được. `no-auth`: không dùng CHAP.
- **ALUA** cho phép target báo trạng thái từng đường (active/optimized, non-optimized, standby) để initiator chọn path. Lab chỉ có 1 port group nên đây chỉ là nhãn, `prio=50` cho biết đường đang active/optimized.

**Ba loại backstore và đường I/O của chúng**:

|Loại|Đường I/O|Đặc điểm|
|---|---|---|
|`block`|Gửi bio thẳng xuống block device|Ít lớp nhất, gần hiệu năng đĩa thật.|
|`fileio`|Đi qua VFS và filesystem của host|Thêm lớp filesystem và page cache. Với `write_back=false` (write-thru), file mở với `O_DSYNC`: mỗi lệnh ghi phải xuống đĩa của host rồi mới trả `GOOD`. Với `write_back=true`, ghi vào page cache rồi trả `GOOD` sớm, nhanh hơn nhưng mất dữ liệu nếu storage host sập trước khi flush.|
|`ramdisk`|Bộ nhớ|Không có đĩa thật, dùng để đo trần của giao thức và mạng.|

Đọc từ backstore `fileio` vẫn có thể được phục vụ từ page cache của storage host.

## 7. Phía initiator Linux và device-mapper multipath

**Thành phần initiator**: `iscsid` (daemon user-space: login, giữ session, NOP, phục hồi), `iscsiadm` (CLI và quản lý node DB), kernel `scsi_transport_iscsi`, `libiscsi`, `iscsi_tcp` (gửi/nhận PDU qua socket TCP). udev tạo `/dev/disk/by-path/ip-10.11.4.21:3260-iscsi-<iqn>-lun-0`, tên này dựa trên **đường đi** (portal, IQN, LUN). Còn `by-id` dựa trên **WWID**.

**Vì sao không dùng thẳng `/dev/sdX`**: tên cấp theo thứ tự phát hiện nên có thể đổi sau reboot, và nếu có 2 đường thì ra 2 `sdX` trỏ cùng dữ liệu.

**device-mapper** là framework kernel tạo block device ảo `/dev/dm-N` từ một bảng ánh xạ gồm các "target" (`linear`, `striped`, `multipath`, `crypt`, `thin`...). LVM cũng dùng dm. Multipath dùng dm target `multipath`.

**Cách multipath hoạt động**:

1. Gom các path cùng WWID (lấy từ VPD 0x83).
2. **Path checker** (mặc định `tur`, tức `TEST UNIT READY` định kỳ) đánh dấu path `active ready` hay `failed`.
3. **Path selector** `service-time 0` chọn path theo thời gian phục vụ ước tính.
4. `hwhandler=alua` để phối hợp với ALUA của target.
5. `features='1 queue_if_no_path'` như đã nói ở mục 5.5.
6. `/etc/multipath/wwids` là danh sách WWID đã từng được multipath.
7. `find_multipaths yes` chỉ tạo map cho đĩa có từ 2 path trở lên hoặc WWID đã có trong `wwids`. Lab chỉ có 1 path nên phải `multipath -a` để đăng ký.
8. `blacklist` loại thiết bị ra khỏi multipath.

**Hiểu lại lỗi đã gặp**: khi ghi `devnode "^sdb$"` vào `blacklist`, multipath bỏ qua chính đĩa iSCSI nên không tạo `mpatha`. Blacklist đúng là đĩa OS (`sda`), thứ multipath không được đụng vào. Lab dùng 1 path nhưng vẫn qua multipath để có **tên ổn định** `/dev/mapper/mpatha` và sẵn sàng nâng lên 2 path.

## 8. Ảo hóa: libvirt, QEMU và virtio-blk

```xml
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native'/>
  <source dev='/dev/mapper/mpatha'/>
  <target dev='vdb' bus='virtio'/>
</disk>
```

- `type='block'` + `source dev`: QEMU mở thẳng block device. Không có định dạng qcow2 nên `type='raw'`. libvirt tự đổi owner và cấp quyền AppArmor cho thiết bị khi VM chạy.
- `cache='none'`: QEMU mở với `O_DIRECT`, bỏ qua page cache của host nên không cache hai lần và độ trễ đo đúng đường thật. Guest vẫn thấy đĩa có write cache và có thể gửi flush, host chuyển thành `SYNCHRONIZE CACHE` xuống iSCSI.
- `io='native'`: dùng Linux AIO (`io_submit`) thay cho thread pool (`io='threads'`), ít overhead. Hợp với `O_DIRECT` nên hay đi cặp với `cache='none'`.

**virtio-blk** là chuẩn paravirtualization: guest biết mình đang chạy trên ảo hóa nên không cần giả lập controller IDE/SATA. Cơ chế cốt lõi là **virtqueue/vring**, vùng nhớ chia sẻ giữa guest và QEMU gồm 3 phần:

- **Descriptor table**: mô tả các buffer.
- **Available ring**: guest đưa request vào.
- **Used ring**: host trả kết quả.

Luồng: driver `virtio_blk` tạo descriptor chain (header, data, status byte) → đặt vào available ring → **kick** (ghi vào thanh ghi thông báo, KVM chuyển thành `ioeventfd` cho QEMU) → QEMU thực hiện I/O trên host → ghi used ring → bơm ngắt vào guest (`irqfd`). Có thể **multi-queue** (mỗi vCPU một queue) để giảm tranh chấp lock, vì vậy nên cấp 4 vCPU cho 4 job fio.

||virtio-blk (`bus='virtio'`)|virtio-scsi (`bus='scsi'` + controller virtio-scsi)|
|---|---|---|
|Guest thấy|`/dev/vdX`|`/dev/sdX`|
|Mô hình|1 thiết bị một đĩa|1 controller nhiều LUN|
|Tập lệnh|Đọc, ghi, flush (discard/write-zeroes tùy phiên bản QEMU)|Đủ tập lệnh SCSI, persistent reservation, passthrough|
|Overhead|Thấp, đơn giản|Hơi cao hơn|
|Số đĩa|Giới hạn theo slot PCI|Nhiều|

## 9. Hành trình một lệnh ghi 4K từ fio xuống đĩa

1. fio gọi `io_submit` với `O_DIRECT` tới `/dev/vdb`.
2. Guest block layer → `virtio_blk` tạo descriptor → **kick**.
3. QEMU lấy request, gọi `io_submit` tới `/dev/mapper/mpatha` với `O_DIRECT`.
4. dm-multipath chọn path (chỉ có `sdb`), SCSI layer sinh CDB `WRITE(16)`.
5. iSCSI initiator đóng CDB thành **SCSI Command PDU** (kèm immediate data hoặc chờ R2T rồi gửi Data-Out), chạy qua TCP.
6. Qua mạng (mất một RTT), đến `10.11.4.21:3260`. LIO nhận PDU, kiểm tra `CmdSN` nằm trong cửa sổ, chuyển cho `target_core`.
7. Backstore `fileio` ghi vào `/storage/iscsi-backing.img` (với `O_DSYNC` khi write-thru), qua ext4 xuống `sda`. Xong mới tạo **SCSI Response `GOOD`**.
8. Response PDU về initiator → SCSI command hoàn tất → dm → QEMU nhận completion → ghi used ring, bơm ngắt → guest → fio ghi nhận hoàn thành.

Độ trễ của một lệnh là tổng các bước trên. Bước 7 (ghi đồng bộ xuống đĩa) thường là bước lớn nhất.

## 10. Áp dụng vào số liệu bạn đã đo

- **Mạng trên com07**: ping trung bình 0.038 ms, iperf3 8.61 Gbps một luồng và 9.47 Gbps bốn luồng. Đây là đường 10 GbE, nên iSCSI không bị giới hạn ~1 Gbps mỗi luồng như lần đo trên mkolla00.
- **seqread 1077 MiB/s** ≈ 9.0 Gbit/s, sát trần TCP 9.47 Gbps. Đọc tuần tự bị chặn bởi **mạng**, không phải bởi đĩa hay giao thức.
- **Định luật Little** (độ trễ trung bình ≈ số I/O đang bay / IOPS): đọc 128 / 80.5k ≈ 1.59 ms (P50 đo được 1.53 ms), ghi 128 / 7468 ≈ 17.1 ms (P50 đo được 17 ms). Số liệu nhất quán, và cho thấy ở qd128 latency chủ yếu là **thời gian xếp hàng**, không phải thời gian phục vụ một I/O.
- **Ghi chậm hơn đọc khoảng 11 lần** (7.5k so với 80.5k IOPS, seqwrite 272 MiB/s so với seqread 1077 MiB/s). Nhiều khả năng do `write_back=false`: mỗi lệnh ghi phải qua ext4 và xuống đĩa của storage host rồi mới trả `GOOD`, còn đọc có thể được phục vụ từ page cache của storage host. Nếu đúng như vậy thì con số 80k IOPS đọc phản ánh mạng cộng cache của storage host, chưa phải tốc độ đĩa thật. Cách kiểm chứng: chạy `iostat -x 1` trên storage host trong lúc fio chạy, nếu đọc gần như không thấy I/O xuống `sda` thì đúng là cache.

## 11. Những thứ lab chưa dùng nhưng nên biết

- **CHAP** (một chiều hoặc hai chiều) và mạng storage tách riêng. Lưu lượng iSCSI mặc định không mã hóa.
- **MC/S và MPIO**: MC/S nhiều kết nối trong một session (ít dùng trên Linux), MPIO nhiều session qua nhiều đường, gộp bằng dm-multipath. Linux dùng MPIO.
- **Jumbo frame** (MTU 9000) giảm số gói và tải CPU khi băng thông cao. Phải bật đồng bộ trên mọi thiết bị trên đường đi.
- **Giao thức thay thế**: iSER (iSCSI qua RDMA), NVMe-oF (độ trễ thấp hơn, giảm lớp SCSI), Ceph RBD.

## 12. Liên hệ với OpenStack Cinder

Lab này gần như tái hiện cách Cinder dùng backend LVM/iSCSI: cinder-volume tạo volume và xuất bằng LIO (tương ứng phần A), nova-compute dùng thư viện os-brick để chạy `iscsiadm` discovery/login (thêm multipath nếu bật), chờ thiết bị xuất hiện, rồi libvirt gắn vào VM bằng đúng kiểu `<disk type='block'>` với `cache='none'`, `io='native'`. Khi có multipath, nguồn thường là `/dev/mapper/<WWID>` thay vì `/dev/mapper/mpatha`. Bạn có thể dùng lab này làm nền để đọc log và code của Cinder sau này.