# Lab: iSCSI Storage → KVM (Multipath) → VM virtio-blk + Benchmark fio

> Mô hình: **Storage Host** export LUN 50G qua iSCSI (LIO/targetcli) → **KVM Host** login iSCSI + bọc bằng device-mapper multipath (`/dev/mapper/mpatha`) → gắn vào **VM** qua **virtio-blk** (`/dev/vdb`) → đo hiệu năng bằng **fio**.

---

## Mục lục

0. [Tổng quan & tham số](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#0-t%E1%BB%95ng-quan--tham-s%E1%BB%91)
1. [Chuẩn bị: fix proxy](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#1-chu%E1%BA%A9n-b%E1%BB%8B-fix-proxy)
2. [Storage Host: tạo LUN 50G và export iSCSI](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#2-storage-host-t%E1%BA%A1o-lun-50g-v%C3%A0-export-iscsi)
3. [KVM Host: lấy InitiatorName và cấp ACL](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#3-kvm-host-l%E1%BA%A5y-initiatorname-v%C3%A0-c%E1%BA%A5p-acl)
4. [Network baseline](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#4-network-baseline)
5. [KVM Host: login iSCSI và cấu hình multipath](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#5-kvm-host-login-iscsi-v%C3%A0-c%E1%BA%A5u-h%C3%ACnh-multipath)
6. [Chuẩn bị máy ảo](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#6-chu%E1%BA%A9n-b%E1%BB%8B-m%C3%A1y-%E1%BA%A3o)
7. [Gắn đĩa vào VM qua virtio-blk](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#7-g%E1%BA%AFn-%C4%91%C4%A9a-v%C3%A0o-vm-qua-virtio-blk)
8. [Benchmark fio trong guest](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#8-benchmark-fio-trong-guest)
9. [Trích xuất số liệu](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#9-tr%C3%ADch-xu%E1%BA%A5t-s%E1%BB%91-li%E1%BB%87u)
10. [Kết quả](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#10-k%E1%BA%BFt-qu%E1%BA%A3)
11. [Nhận xét nhanh](https://claude.ai/chat/4e334147-ec32-4afb-8f6e-0a2bd1055e62#11-nh%E1%BA%ADn-x%C3%A9t-nhanh)

---

## 0. Tổng quan & tham số

|Thành phần|Giá trị|
|---|---|
|Storage Host|`10.11.4.21` (cổng iSCSI `3260`)|
|KVM Host (`com07`)|`vhhl1c2lab2com07` – `10.11.4.23` (đã có sẵn libvirt và VM `vm01`)|
|Target IQN|`iqn.2026-09.local.fptcloud:storage.target1`|
|Backstore|`fileio` – `disk1` – `/var/tmp/disk1.img` – 50G|
|LUN WWID|`36001405b94ac0b447ed40e5a150b654e`|
|Thiết bị trên KVM Host|`/dev/sdb` → `/dev/mapper/mpatha` (`dm-0`)|
|VM|`vm01` (có sẵn) hoặc `test-vm` (tạo mới từ cloud image)|
|Thiết bị trong VM|`/dev/vdb` (virtio-blk, raw, `cache=none`, `io=native`)|

**Luồng dữ liệu:**

```text
fio (guest) → /dev/vdb (virtio-blk) → /dev/mapper/mpatha (dm-multipath)
            → /dev/sdb (iSCSI initiator) → TCP 10.11.4.21:3260 → LIO target → disk1.img
```

---

## 1. Chuẩn bị: fix proxy

Nếu `apt` / `wget` bị lỗi khi tải gói, kiểm tra biến proxy rồi gắn proxy cho `apt` trên **cả máy ảo lẫn compute host**:

```bash
env | grep -i proxy
```

---

## 2. Storage Host: tạo LUN 50G và export iSCSI

_Chạy trên **Storage Host** (`10.11.4.21`):_

```bash
sudo targetcli
```

Trong giao diện `targetcli`:

```text
# 1. Tạo backstore 50G dạng file
cd /backstores/fileio
create disk1 /var/tmp/disk1.img 50G

# 2. Tạo target iSCSI
cd /iscsi
create iqn.2026-09.local.fptcloud:storage.target1

# 3. Gắn đĩa vào target (LUN 0)
cd /iscsi/iqn.2026-09.local.fptcloud:storage.target1/tpg1/luns
create /backstores/fileio/disk1

# 4. ACL cho KVM Host: làm ở mục 3
saveconfig
exit
```

> Có thể dùng `/backstores/block` thay cho `fileio` nếu dùng phân vùng / LVM thô.

---

## 3. KVM Host: lấy InitiatorName và cấp ACL

### 3.1. Lấy InitiatorName của com07

_Chạy trên **com07** (`10.11.4.23`):_

```bash
cat /etc/iscsi/initiatorname.iscsi
```

Copy chuỗi sau dấu `=` (dạng `iqn.2004-10.com.ubuntu:01:...`), gọi tạm là `<IQN_COM07>`.

### 3.2. Cập nhật ACL trên Storage Host

_Chạy trên **Storage Host**:_

```bash
sudo targetcli
```

```text
cd /iscsi/iqn.2026-09.local.fptcloud:storage.target1/tpg1/acls
create <IQN_COM07>
saveconfig
exit
```

> Nếu còn ACL của các máy cũ (`com09`, `mkolla00`) không dùng nữa, xoá bằng `delete <IQN_cũ>` trong thư mục `acls`.

---

## 4. Network baseline

Đo mốc mạng trần trước khi test storage để biết giới hạn của đường truyền.

### 4.1. Ping latency

_Chạy trên **com07**:_

```bash
ping -c 20 10.11.4.21
```

Ghi lại giá trị `avg` ở dòng `rtt min/avg/max/mdev`.

### 4.2. Throughput TCP bằng iperf3

_Trên **Storage Host**, bật server:_

```bash
iperf3 -s
```

_Trên **com07**:_

```bash
# Cài iperf3 nếu chưa có
sudo apt update && sudo apt install -y iperf3

# Test 1 luồng
iperf3 -c 10.11.4.21 -t 30 -i 5

# Test 4 luồng song song
iperf3 -c 10.11.4.21 -t 30 -P 4
```

Xong thì nhấn `Ctrl + C` trên Storage Host để tắt `iperf3 -s`.

**Kết quả đo được (com07 → Storage Host):**

|Test|Kết quả|
|---|---|
|Ping (10 gói)|min/avg/max/mdev = 0.029 / **0.038** / 0.062 / 0.008 ms, 0% loss|
|iperf3 – 1 luồng, 30s|**8.61 Gbit/s** (30.1 GBytes, 856 retransmit)|
|iperf3 – 4 luồng (`-P 4`), 30s|**9.47 Gbit/s** tổng (~2.37 Gbit/s mỗi luồng, 0 retransmit)|

---

## 5. KVM Host: login iSCSI và cấu hình multipath

_Chạy toàn bộ trên **com07** (`10.11.4.23`)._

### 5.1. Cài công cụ

```bash
sudo apt install -y open-iscsi multipath-tools
sudo systemctl enable --now iscsid
sudo systemctl enable --now multipathd
```

### 5.2. Discovery và login

```bash
# Quét target
sudo iscsiadm -m discovery -t sendtargets -p 10.11.4.21:3260

# Login vào target
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 --login

# Tự động kết nối lại khi reboot
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 \
  -o update -n node.startup -v automatic

# Kiểm tra ổ đĩa nhận diện
lsblk
```

Xác nhận xuất hiện ổ 50G mới (ví dụ `/dev/sdb`).

### 5.3. Cấu hình multipath để bọc LUN

**Bước 1 – Lấy WWID và đăng ký vào multipath:**

```bash
# Lấy WWID của /dev/sdb (nếu chưa biết)
sudo /lib/udev/scsi_id -g -u -d /dev/sdb

# Đăng ký WWID
sudo multipath -a 36001405b94ac0b447ed40e5a150b654e
# → wwid '36001405b94ac0b447ed40e5a150b654e' added
```

**Bước 2 – Tạo `/etc/multipath.conf`** (chặn ổ OS, chỉ cho phép LUN iSCSI):

```bash
sudo tee /etc/multipath.conf << 'EOF'
defaults {
    user_friendly_names yes
    find_multipaths yes
}

blacklist {
    # Chặn ổ đĩa OS (sda và các phân vùng sda1, sda2...)
    devnode "^sda[0-9]*"
    # Chặn các ổ ảo snap/loop
    devnode "^loop[0-9]*"
}

blacklist_exceptions {
    # Luôn cho phép LUN iSCSI này được tạo map
    wwid "36001405b94ac0b447ed40e5a150b654e"
}
EOF
```

**Bước 3 – Reload và kiểm tra:**

```bash
sudo systemctl restart multipathd
sudo multipath -r
sudo multipath -ll
ls -l /dev/mapper/
```

Kết quả mong đợi: `mpatha` (50G) bọc `sdb`, trỏ tới `dm-0` (hoặc `dm-X`).

> **Lưu ý:** blacklist phải chặn đúng ổ OS (`sda`), **không** chặn `sdb` (LUN iSCSI), nếu không `mpatha` sẽ không xuất hiện.

---

## 6. Chuẩn bị máy ảo

Có hai lựa chọn.

### Cách A – Dùng VM có sẵn `vm01`

`vm01` đã có sẵn trên com07 (trạng thái `shut off`), bỏ qua phần tạo VM và sang mục 7.

### Cách B – Tạo `test-vm` từ Ubuntu Cloud Image (nhanh nhất)

Tải Ubuntu Cloud Image và dùng `cloud-init` để set sẵn mật khẩu cho user `ubuntu`.

**1. Tải image và tạo ổ đĩa OS 20G:**

```bash
sudo mkdir -p /var/lib/libvirt/images
cd /var/lib/libvirt/images

# Tải Ubuntu 22.04 Cloud Image (hoặc copy từ máy khác nếu com07 không tải được)
sudo wget -O ubuntu-22.04.qcow2 \
  https://cloud-images.ubuntu.com/jammy/current/jammy-server-cloudimg-amd64.img

# Tạo ổ OS 20G dựa trên image gốc
sudo qemu-img create -f qcow2 -F qcow2 \
  -b /var/lib/libvirt/images/ubuntu-22.04.qcow2 \
  /var/lib/libvirt/images/test-vm-root.qcow2 20G
```

**2. Tạo file cloud-init để đặt mật khẩu SSH** _(chỉ dùng cho lab)_:

```bash
cat > /tmp/user-data << 'EOF'
#cloud-config
password: 123456
chpasswd: { expire: False }
ssh_pwauth: True
EOF

# Tạo đĩa config
sudo cloud-localds /var/lib/libvirt/images/test-vm-seed.iso /tmp/user-data
```

**3. Tạo VM:**

```bash
sudo virt-install \
  --name test-vm \
  --memory 4096 \
  --vcpus 4 \
  --disk path=/var/lib/libvirt/images/test-vm-root.qcow2,format=qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/test-vm-seed.iso,device=cdrom \
  --os-variant generic \
  --network network=default \
  --graphics none \
  --import \
  --noautoconsole
```

---

## 7. Gắn đĩa vào VM qua virtio-blk

_Chạy trên **com07**._

### 7.1. Tạo file XML mô tả ổ đĩa

```bash
cat > /tmp/disk-iscsi.xml << 'EOF'
<disk type='block' device='disk'>
  <driver name='qemu' type='raw' cache='none' io='native'/>
  <source dev='/dev/mapper/mpatha'/>
  <target dev='vdb' bus='virtio'/>
</disk>
EOF
```

### 7.2. Attach vào VM

**Với `vm01`** (đang `shut off`) – gắn vào cấu hình rồi bật máy:

```bash
sudo virsh attach-device vm01 /tmp/disk-iscsi.xml --config
sudo virsh start vm01
sudo virsh dumpxml vm01 | grep -A 5 "vdb"
```

**Với `test-vm`** (đang chạy) – gắn cả cấu hình lẫn live:

```bash
sudo virsh attach-device test-vm /tmp/disk-iscsi.xml --config --live
sudo virsh dumpxml test-vm | grep -A 5 "vdb"
```

---

## 8. Benchmark fio trong guest

Đăng nhập vào máy ảo qua console hoặc SSH:

```bash
virsh console vm01      # hoặc test-vm
```

### 8.1. Kiểm tra ổ đĩa và cài fio

```bash
lsblk
sudo apt update && sudo apt install -y fio
```

Xác nhận thấy `/dev/vdb` dung lượng **50G**.

> ⚠️ **Giữ nguyên raw block, không format filesystem.** Các test ghi (`randwrite`, `write`, `randrw`) ghi trực tiếp lên `/dev/vdb` và sẽ **phá dữ liệu** trên đĩa này.

### 8.2. Các test case

Tất cả dùng chung: `--direct=1 --ioengine=libaio --time_based --runtime=60 --ramp_time=10 --group_reporting --lat_percentiles=1`.

**Test 1 – Random Read IOPS (4K, QD32, 4 jobs):**

```bash
sudo fio --name=randread_test \
  --filename=/dev/vdb \
  --direct=1 \
  --rw=randread \
  --bs=4k \
  --iodepth=32 \
  --numjobs=4 \
  --ioengine=libaio \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting \
  --lat_percentiles=1 \
  --output=/tmp/result_randread.log
```

**Test 2 – Random Write IOPS (4K, QD32, 4 jobs):**

```bash
sudo fio --name=randwrite_test \
  --filename=/dev/vdb \
  --direct=1 \
  --rw=randwrite \
  --bs=4k \
  --iodepth=32 \
  --numjobs=4 \
  --ioengine=libaio \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting \
  --lat_percentiles=1 \
  --output=/tmp/result_randwrite.log
```

**Test 3 – Sequential Read băng thông (1M, QD8, 1 job):**

```bash
sudo fio --name=seqread_test \
  --filename=/dev/vdb \
  --direct=1 \
  --rw=read \
  --bs=1M \
  --iodepth=8 \
  --numjobs=1 \
  --ioengine=libaio \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting \
  --lat_percentiles=1 \
  --output=/tmp/result_seqread.log
```

**Test 4 – Sequential Write băng thông (1M, QD8, 1 job):**

```bash
sudo fio --name=seqwrite_test \
  --filename=/dev/vdb \
  --direct=1 \
  --rw=write \
  --bs=1M \
  --iodepth=8 \
  --numjobs=1 \
  --ioengine=libaio \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting \
  --lat_percentiles=1 \
  --output=/tmp/result_seqwrite.log
```

**Test 5 – Mixed 70/30 đọc/ghi (mô phỏng database):**

```bash
sudo fio --name=mixed_test \
  --filename=/dev/vdb \
  --direct=1 \
  --rw=randrw --rwmixread=70 \
  --bs=4k \
  --iodepth=32 \
  --numjobs=4 \
  --ioengine=libaio \
  --time_based --runtime=60 --ramp_time=10 \
  --group_reporting \
  --lat_percentiles=1 \
  --output=/tmp/result_mixed.log
```

---

## 9. Trích xuất số liệu

Chạy trong guest (từng lệnh riêng, tránh dán chồng lên nhau):

```bash
grep -E "IOPS|BW|clat percentiles|50.00th|99.00th" /tmp/result_randread.log
grep -E "IOPS|BW|clat percentiles|50.00th|99.00th" /tmp/result_randwrite.log
grep -E "IOPS|BW|clat percentiles|50.00th|99.00th" /tmp/result_seqread.log
grep -E "IOPS|BW|clat percentiles|50.00th|99.00th" /tmp/result_seqwrite.log
grep -E "IOPS|BW|clat percentiles|50.00th|99.00th" /tmp/result_mixed.log
```

> **Đọc output:** vì bật `--lat_percentiles=1`, mỗi test in **2 khối percentile**: khối đầu là `clat` (completion latency), khối sau là `lat` (tổng latency). Bảng bên dưới dùng số **clat**. Lưu ý đơn vị: `usec` hoặc `msec` ghi ở dòng `clat percentiles`.

---

## 10. Kết quả

### 10.1. Network baseline (com07 → Storage Host)

|Chỉ số|Giá trị|
|---|---|
|Ping avg|0.038 ms|
|iperf3 1 luồng|8.61 Gbit/s|
|iperf3 4 luồng|9.47 Gbit/s|

### 10.2. fio trên `/dev/vdb` (virtio-blk ← mpatha ← iSCSI)

|Test case|Block size / QD|IOPS|Bandwidth|P50 latency (clat)|P99 latency (clat)|
|---|---|---|---|---|---|
|**Random Read**|4K / QD32 / 4 jobs|**80.5k**|315 MiB/s (330 MB/s)|1.53 ms|3.82 ms|
|**Random Write**|4K / QD32 / 4 jobs|**7,468**|29.2 MiB/s (30.6 MB/s)|17 ms|47 ms|
|**Sequential Read**|1M / QD8 / 1 job|1,076|**1,077 MiB/s (1,129 MB/s)**|–|–|
|**Sequential Write**|1M / QD8 / 1 job|272|**272 MiB/s (285 MB/s)**|–|–|
|**Mixed 70/30 – Read**|4K / QD32 / 4 jobs|15.4k|60.1 MiB/s (63.0 MB/s)|4.75 ms|16.6 ms|
|**Mixed 70/30 – Write**|4K / QD32 / 4 jobs|6,615|25.8 MiB/s (27.1 MB/s)|4.42 ms|18.7 ms|

> Dấu “–”: chưa trích latency cho hai test sequential (lệnh grep ban đầu chỉ lấy `IOPS|BW`). Muốn điền thì chạy lại lệnh ở mục 9 cho `result_seqread.log` và `result_seqwrite.log`.

---

## 11. Nhận xét nhanh

- **Sequential Read ≈ 1,129 MB/s ≈ 9.0 Gbit/s**, sát với trần mạng đo bằng iperf3 (~9.47 Gbit/s với 4 luồng) → đọc tuần tự bị giới hạn bởi **đường mạng**, không phải bởi storage.
- **Random Read 80.5k IOPS** với P99 chỉ ~3.8 ms là mức cao; khả năng cao phần lớn đọc được phục vụ từ page cache của Storage Host (backstore là file `disk1.img`), nên nên đối chiếu thêm với chế độ không cache nếu cần số liệu "thật" của đĩa.
- **Random Write chỉ ~7.5k IOPS, P50 17 ms** thấp hơn hẳn đọc: đây là điểm nghẽn ghi rõ nhất. Cần kiểm tra chế độ ghi của backstore trên Storage Host (write-thru hay write-back) và tốc độ đĩa vật lý phía sau file `disk1.img`.
- Các kết luận trên là suy đoán từ số liệu, chưa được kiểm chứng bằng đo thêm (ví dụ: chạy fio thẳng trên Storage Host, hoặc đổi `write_back` của backstore).

---

## Phụ lục: lỗi thường gặp

|Triệu chứng|Nguyên nhân / cách xử lý|
|---|---|
|`apt` / `wget` không tải được|Thiếu proxy → `env \| grep -i proxy`, cấu hình proxy cho `apt` trên cả VM và compute host|
|`/dev/mapper/mpatha` không xuất hiện|Blacklist nhầm ổ; chỉ blacklist `sda` và `loop`, thêm `wwid` của LUN vào `blacklist_exceptions`, rồi `multipath -r`|
|Login iSCSI bị từ chối|Chưa tạo ACL với đúng `InitiatorName` của KVM Host trên Storage Host|
|`iperf3` không kết nối được|Chưa chạy `iperf3 -s` trên Storage Host|
|`grep: ... No such file or directory`|File log của test đó chưa được tạo → chạy test tương ứng trước (ví dụ `seqwrite`)|



---
Boot từ iscsi
```

# 1. Tắt và xóa định nghĩa máy ảo vm01 cũ để nhả ổ mapper
sudo virsh destroy vm01 2>/dev/null || true
sudo virsh undefine vm01 2>/dev/null || true

# 2. Chạy lại lệnh tạo VM boot từ iSCSI (thêm cờ bỏ qua kiểm tra trùng lặp)
sudo virt-install \
  --name test-vm-iscsi-boot \
  --memory 4096 \
  --vcpus 4 \
  --disk path=/dev/mapper/mpatha,bus=virtio,cache=none,io=native \
  --disk path=/var/lib/libvirt/images/test-vm-seed.iso,device=cdrom \
  --os-variant generic \
  --network network=default \
  --graphics none \
  --boot hd \
  --check path_in_use=off \
  --noautoconsole
# Chạy bài test mô phỏng tải đọc/ghi ngẫu nhiên (4K, Queue Depth 32, 4 luồng)
sudo fio --name=boot_disk_bench \
  --filename=/tmp/fio_testfile \
  --size=5G \
  --direct=1 \
  --rw=randrw \
  --rwmixread=70 \
  --bs=4k \
  --iodepth=32 \
  --numjobs=4 \
  --ioengine=libaio \
  --time_based --runtime=60 \
  --group_reporting \
  --output=/tmp/result_fio.log
ubuntu@ubuntu:~$ grep -E "IOPS|BW|clat" /tmp/result_fio.log7k,w=16.7k IOPS][eta 00m:00s]
  read: IOPS=38.6k, BW=151MiB/s (158MB/s)(9037MiB/60003msec)
    clat (usec): min=71, max=12890, avg=2375.37, stdev=1454.61
    clat percentiles (usec):
  write: IOPS=16.6k, BW=64.7MiB/s (67.9MB/s)(3883MiB/60003msec); 0 zone resets
    clat (usec): min=142, max=12215, avg=2176.45, stdev=1481.06
    clat percentiles (usec):

```