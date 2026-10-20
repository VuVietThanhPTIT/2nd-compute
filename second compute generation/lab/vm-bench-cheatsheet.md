# Cheatsheet: VM benchmark iSCSI (KVM/libvirt) + lệnh virsh hay dùng

Mô hình: host (com08/com09) thấy LUN iSCSI qua multipath (`/dev/mapper/mpathb`), gắn nguyên LUN vào VM dưới dạng `/dev/vdb` (raw, `cache=none`, `io=native`), rồi chạy `fio` trong VM.

> **Cảnh báo**
> - Test ghi **xóa sạch dữ liệu** trên LUN `mpathb`.
> - LUN này **dùng chung** cho nhiều host (cùng WWID). Chỉ chạy bench ở **một nơi tại một thời điểm**.
> - Mở terminal mới thì phải khai lại biến ở Bước 1.

---

## Bước 0. Kiểm tra host (không cài lại nếu đã đủ)

```bash
command -v virsh virt-install qemu-img cloud-localds \
  || echo "thiếu gói: sudo apt install -y libvirt-clients virtinst qemu-utils cloud-image-utils"
systemctl is-active dbus libvirtd systemd-machined     # phải ra 3 dòng active
ls -l /dev/kvm
virsh net-list                                         # default phải active
virsh net-start default 2>/dev/null || true
ls -lh /var/lib/libvirt/images/noble-base.img          # base image
df -h /var                                             # còn đủ dung lượng
```

Kiểm tra systemd có tạo được scope không (thao tác libvirt cần khi start VM). Nếu lệnh này treo thì `virsh start` cũng sẽ treo:

```bash
timeout 40 systemd-run --scope --quiet true; echo "scope exit=$?"    # phải exit=0
```

---

## Bước 1. Khai biến và tạo đĩa OS

```bash
export IMG=/var/lib/libvirt/images
export VM=vm1
export BASE=$IMG/noble-base.img
export PROXY=http://10.10.2.22:3001

# Host tải gói/image qua proxy (chỉ cần khi phải wget)
export HTTP_PROXY=$PROXY HTTPS_PROXY=$PROXY http_proxy=$PROXY https_proxy=$PROXY


[ -f noble-base.img ] || sudo wget -O noble-base.img \
  https://cloud-images.ubuntu.com/noble/current/noble-server-cloudimg-amd64.img
sudo cp noble-base.img $VM.qcow2
sudo qemu-img resize $VM.qcow2 +10G


```

---

## Bước 2. Cloud-init (proxy cho apt, biến proxy, tool benchmark)

`EOF` **không có nháy đơn** để `$VM` và `$PROXY` được thay giá trị.

```bash
cat > /tmp/$VM-user-data << EOF
#cloud-config
hostname: $VM
password: 123456
chpasswd: { expire: false }
ssh_pwauth: true

# Proxy cho apt (áp dụng trước khi cài gói)
apt:
  http_proxy: $PROXY
  https_proxy: $PROXY

# Proxy cho shell và các tool khác (login mới đều có)
write_files:
  - path: /etc/environment
    append: true
    content: |
      HTTP_PROXY=$PROXY
      HTTPS_PROXY=$PROXY
      http_proxy=$PROXY
      https_proxy=$PROXY
      NO_PROXY=localhost,127.0.0.1,10.0.0.0/8
      no_proxy=localhost,127.0.0.1,10.0.0.0/8
  - path: /etc/apt/apt.conf.d/99proxy
    content: |
      Acquire::http::Proxy "$PROXY";
      Acquire::https::Proxy "$PROXY";

package_update: true
packages:
  - fio
  - jq
  - bsdextrautils
  - sysstat
  - iotop
  - htop
  - qemu-guest-agent

runcmd:
  - systemctl enable --now qemu-guest-agent
  - touch /var/lib/cloud/bench-tools-done
EOF

printf "instance-id: %s\nlocal-hostname: %s\n" $VM $VM > /tmp/$VM-meta-data
sudo cloud-localds $IMG/$VM-seed.img /tmp/$VM-user-data /tmp/$VM-meta-data

grep -A2 '^apt:' /tmp/$VM-user-data      # kiểm tra proxy đã được thay giá trị
```

---

## Bước 3. Kiểm tra `mpathb` trước khi gắn vào VM

```bash
multipath -ll mpathb                     # path active ready, size ~447G
lsblk /dev/mapper/mpathb                 # cột MOUNTPOINTS phải trống
findmnt /dev/mapper/mpathb               # không in gì = chưa mount
timeout 10 dd if=/dev/mapper/mpathb of=/dev/null bs=1M count=1 iflag=direct; echo "dd exit=$?"

# Không VM nào đang giữ mpathb
for d in $(virsh list --all --name); do
  virsh domblklist $d | grep -q mpathb && echo "VM $d đang dùng mpathb"
done
```

---

## Bước 4. Tạo VM với đĩa raw để bench

```bash
sudo virt-install \
  --name $VM --ram 4096 --vcpus 4 \
  --disk path=$IMG/$VM.qcow2,format=qcow2,bus=virtio \
  --disk path=/dev/mapper/mpathb,format=raw,bus=virtio,cache=none,io=native,serial=benchdisk \
  --disk path=$IMG/$VM-seed.img,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics none --import --noautoconsole
```

| Tham số đĩa bench | Tác dụng |
|---|---|
| `format=raw` | Bắt buộc với block device, không có lớp qcow2 chen vào |
| `bus=virtio` | VM thấy đĩa là `/dev/vdb` |
| `cache=none` | qemu mở đĩa bằng `O_DIRECT`, bỏ qua page cache của host |
| `io=native` | Dùng Linux AIO thay cho thread pool. Luôn đi cặp với `cache=none` |
| `serial=benchdisk` | Gán số serial để nhận diện đĩa trong VM, tránh nhầm với `vda` |

Nếu `virt-install` báo lỗi OS variant, dùng `--os-variant detect=on,require=off`.

Nếu `virt-install` hoặc `virsh start` bị treo, **đừng Ctrl+C**. Tách thành 3 bước để biết chính xác chỗ treo:

```bash
sudo virt-install ...các tham số như trên... --print-xml > /tmp/$VM.xml
ls -l /tmp/$VM.xml                        # phải > 0 byte, nếu 0 byte thì KHÔNG define
virsh define /tmp/$VM.xml
timeout 90 virsh start $VM; echo "start exit=$?"
```

### Xác nhận cấu hình đã đúng

```bash
virsh domblklist $VM                                     # vdb -> /dev/mapper/mpathb
virsh dumpxml $VM | grep -B1 -A4 mpathb
virsh dumpxml $VM | grep "driver name='qemu'"            # cache='none' io='native'
```

Dòng driver của đĩa mpathb phải là:

```
<driver name='qemu' type='raw' cache='none' io='native'/>
```

---

## Bước 5. Lấy địa chỉ IP của VM

Chạy trên host:

```bash
virsh domifaddr $VM                      # cách nhanh nhất
virsh net-dhcp-leases default            # xem DHCP lease: hostname, MAC, IP

# Lấy riêng IP để dùng cho ssh
IP=$(virsh domifaddr $VM | awk '/ipv4/ {print $4}' | cut -d/ -f1); echo $IP
```

Nếu trống (VM mới bật cần khoảng 30-60 giây):

```bash
virsh domifaddr $VM --source agent       # cần qemu-guest-agent đã chạy trong VM
virsh domifaddr $VM --source arp         # dò từ bảng ARP
virsh domiflist $VM                      # xem MAC
ip neigh | grep -i <MAC>
virsh console $VM                        # đăng nhập, chạy: hostname -I   (thoát: Ctrl+])
```

---

## Bước 6. Vào VM và kiểm tra (đợi 1-2 phút cho cloud-init)

```bash
sleep 60
ssh ubuntu@$IP                           # mật khẩu 123456
```

Trong VM:

```bash
cloud-init status --wait                 # phải ra "status: done"
command -v fio jq column iostat          # đủ tool
env | grep -i proxy                      # có các biến proxy
cat /etc/apt/apt.conf.d/99proxy
lsblk -o NAME,SIZE,TYPE,SERIAL,MOUNTPOINTS
# vda ~12G (có mount), vdb ~447G, SERIAL=benchdisk, không mount
ls -l /dev/disk/by-id/ | grep benchdisk
```

Nếu `apt` không tải được gói:

```bash
grep -i proxy /var/log/cloud-init.log
curl -I -x $http_proxy http://archive.ubuntu.com
```

---

## Bước 7. Copy script và chạy bench

Từ host, copy script vào VM (nên dùng `scp`, đừng dán qua Notepad/Word vì sẽ mất ký tự `$` và `#`):

```bash
scp bench.sh ubuntu@$IP:~/
ssh ubuntu@$IP
```

Trong VM, chạy trong `tmux` vì mất khoảng 30 phút:

```bash
tmux new -s bench
sudo env SEQ=1 ./bench.sh /dev/vdb vm_vdb 60
```

Chạy thử nhanh trước:

```bash
sudo env QDS="1 8" ./bench.sh /dev/vdb test_nhanh 20
```

| Tham số | Ý nghĩa |
|---|---|
| `SEQ=1` | Bật thêm 2 lượt tuần tự 1M (`seqread`, `seqwrite`) |
| `$1` = `/dev/vdb` | **Thiết bị bị đo và bị ghi đè**. Trong VM là `vdb`, không phải `sdb` |
| `$2` = `vm_vdb` | Nhãn, kết quả vào `./result_vm_vdb/` |
| `$3` = `60` | Số giây mỗi lượt (cộng thêm 10 giây ramp, không tính vào kết quả) |
| `QDS="1 8 32"` | Chọn các mức queue depth (mặc định `1 2 4 8 16 32 64 128`) |

Trước khi gõ `YES`, đọc dòng `Test GHI se PHA du lieu tren ...`: phải là `/dev/vdb`. Nếu là `/dev/vda` thì dừng ngay vì đó là đĩa OS.

Kết quả: `./result_<tag>/summary.txt` và các file JSON từng lượt. Lấy về host:

```bash
scp -r ubuntu@$IP:~/result_vm_vdb .
```

### Nội dung `bench.sh`

```bash
cat > bench.sh << 'EOF'
#!/bin/bash
# Usage: sudo [QDS="1 4 16"] [SEQ=1] ./bench.sh <device> <tag> [runtime_sec]
# VD: sudo ./bench.sh /dev/vdb vm_vdb 60
set -u
DEV=${1:?thieu device, vd /dev/vdb}
TAG=${2:?thieu tag, vd vm_vdb}
RT=${3:-60}
QDS=${QDS:-"1 2 4 8 16 32 64 128"}   # cac muc queue depth can do
BS=4k                                # block size chuan
REAL=$(readlink -f "$DEV")

if lsblk -nro MOUNTPOINT "$REAL" 2>/dev/null | grep -q . ; then
  echo "ABORT: $REAL (hoac phan vung con) dang duoc mount"; exit 1
fi
read -r -p "Test GHI se PHA du lieu tren $REAL. Go YES de tiep tuc: " ANS
[ "$ANS" = "YES" ] || { echo "Huy."; exit 1; }

OUT=./result_$TAG; mkdir -p "$OUT"

# direct=1: bo qua page cache; sync=0: khong O_SYNC; libaio: tuong duong io=native
COMMON=(--filename="$REAL" --size=50G --direct=1 --sync=0 --ioengine=libaio
        --time_based --runtime="$RT" --ramp_time=10 --group_reporting
        --clat_percentiles=1 --percentile_list=99:99.99 --output-format=json)

run() { local n=$1; shift; echo ">>> $n"; fio --name="$n" "${COMMON[@]}" "$@" --output="$OUT/$n.json"; }

# numjobs=1 => QD = so I/O dang cho. QD=1 la phep do latency thuan.
for QD in $QDS; do
  run randread_qd$QD  --rw=randread  --bs=$BS --iodepth=$QD --numjobs=1
  run randwrite_qd$QD --rw=randwrite --bs=$BS --iodepth=$QD --numjobs=1
  run mixed_qd$QD     --rw=randrw --rwmixread=70 --bs=$BS --iodepth=$QD --numjobs=1
done

# Tuy chon: bang thong tuan tu 1M, QD8 (bat bang SEQ=1)
if [ "${SEQ:-0}" = "1" ]; then
  run seqread_bs1M  --rw=read  --bs=1M --iodepth=8 --numjobs=1
  run seqwrite_bs1M --rw=write --bs=1M --iodepth=8 --numjobs=1
fi

# Tong hop ket qua
{
  echo -e "test\tparam\tdir\tIOPS\tMiB/s\tavg_ms\tP99_ms\tP99.99_ms"
  for f in $(ls "$OUT"/*.json | sort -V); do
    n=$(basename "$f" .json); t=${n%%_*}; p=${n#*_}
    jq -r --arg t "$t" --arg p "$p" '.jobs[0] | to_entries[]
      | select(.key=="read" or .key=="write") | select(.value.io_bytes>0)
      | [$t, $p, .key, (.value.iops|floor), ((.value.bw/1024)|floor),
         ((((.value.clat_ns.mean // 0)/1e6*1000)|round)/1000),
         ((((.value.clat_ns.percentile["99.000000"] // 0)/1e6*1000)|round)/1000),
         ((((.value.clat_ns.percentile["99.990000"] // 0)/1e6*1000)|round)/1000)]
      | @tsv' "$f"
  done
} | column -t -s $'\t' | tee "$OUT/summary.txt"
EOF
chmod +x bench.sh
bash -n bench.sh && echo "cu phap OK"
grep -c '\$' bench.sh      # phai ra so lon (hang chuc), neu 0 la file da mat dau $
```

Chạy bench ngay trên **host** (đo bare-metal để so sánh): truyền `/dev/mapper/mpathb` thay vì `/dev/sdb`, và phải chắc chắn không VM nào đang giữ `mpathb`:

```bash
sudo env SEQ=1 ./bench.sh /dev/mapper/mpathb host_mpathb 60
```

Dòng cảnh báo sẽ hiện `/dev/dm-N` (do `readlink -f`). Đối chiếu bằng `ls -l /dev/mapper/mpathb`.

---

## Bước 8. Dọn sau khi đo

```bash
virsh destroy $VM                        # tắt cưỡng bức (bỏ qua nếu đã tắt)
virsh undefine $VM                       # xóa định nghĩa, KHÔNG đụng LUN mpathb
rm -f $IMG/$VM.qcow2 $IMG/$VM-seed.img
```

Tránh dùng `--remove-all-storage` khi VM có gắn đĩa thật, để chắc chắn không động vào `mpathb`.

---

## Lệnh virsh hay dùng

### Vòng đời VM

```bash
virsh list --all                         # liệt kê VM (kể cả đang tắt)
virsh start $VM
virsh shutdown $VM                       # tắt mềm (cần guest phản hồi ACPI)
virsh destroy $VM                        # tắt cưỡng bức (như rút điện, không xóa VM)
virsh reboot $VM
virsh suspend $VM; virsh resume $VM      # tạm dừng / tiếp tục
virsh autostart $VM                      # tự chạy khi host boot
virsh autostart --disable $VM
virsh undefine $VM                       # xóa định nghĩa VM
virsh domstate $VM                       # trạng thái
virsh console $VM                        # vào console (thoát: Ctrl+])
```

Hàm đợi VM tắt hẳn (dùng trước khi clone/backup):

```bash
wait_off() { while [ "$(virsh domstate "$1")" != "shut off" ]; do sleep 2; done; }
```

### Thông tin VM

```bash
virsh dominfo $VM                        # CPU, RAM, trạng thái, autostart
virsh dumpxml $VM                        # toàn bộ cấu hình XML
virsh edit $VM                           # sửa XML trực tiếp
virsh domblklist $VM                     # danh sách đĩa
virsh domiflist $VM                      # danh sách card mạng + MAC
virsh domblkstat $VM vdb                 # thống kê I/O của đĩa vdb
virsh domstats $VM --block               # thống kê đĩa chi tiết
virsh vcpuinfo $VM                       # thông tin vCPU
virsh domuuid $VM
```

### Đĩa: gắn / tháo

```bash
# Gắn đĩa mpathb vào VM đang chạy (giữ vĩnh viễn)
virsh attach-disk $VM /dev/mapper/mpathb vdb \
  --driver qemu --subdriver raw --cache none --io native \
  --targetbus virtio --persistent

virsh detach-disk $VM vdb --persistent   # tháo đĩa
virsh domblkinfo $VM vda                 # dung lượng đĩa
```

### Mạng

```bash
virsh net-list --all
virsh net-start default
virsh net-autostart default
virsh net-dhcp-leases default            # IP đã cấp
virsh net-dumpxml default
virsh net-edit default
```

### Snapshot

```bash
virsh snapshot-create-as $VM snap-initial --description "Clean OS"
virsh snapshot-list $VM
virsh snapshot-revert $VM snap-initial
virsh snapshot-delete $VM snap-initial
```

> Snapshot nội bộ không dùng được trên đĩa raw gắn trực tiếp (`mpathb`). Nếu VM có đĩa này, chỉ snapshot đĩa OS hoặc dùng `--disk-only` với `--diskspec`.

### Clone

```bash
virsh shutdown $VM || true; wait_off $VM
sudo virt-clone --original $VM --name $VM-clone --file $IMG/$VM-clone.qcow2
```

Khi clone VM có đĩa `mpathb`, chỉ clone đĩa OS (`--skip-copy vdb`), nếu không `virt-clone` sẽ cố copy 447G.

### Đổi tài nguyên (vCPU, RAM, đĩa)

```bash
virsh setvcpus $VM 4 --config --maximum
virsh setvcpus $VM 4 --config
virsh setmaxmem $VM 4194304 --config     # đơn vị KiB (4194304 KiB = 4 GiB)
virsh setmem $VM 4194304 --config
sudo qemu-img resize $IMG/$VM.qcow2 +10G
virsh start $VM
# Trong guest: nproc ; free -h ; df -h /
```

### Backup / xóa / restore

```bash
mkdir -p ~/vm-backup
virsh shutdown $VM || true; wait_off $VM
virsh dumpxml $VM > ~/vm-backup/$VM.xml
sudo cp --sparse=always $IMG/$VM.qcow2 $IMG/$VM-seed.img ~/vm-backup/

virsh undefine $VM --snapshots-metadata  # xóa VM (giữ file đĩa)

# Restore
sudo cp --sparse=always ~/vm-backup/$VM.qcow2 ~/vm-backup/$VM-seed.img $IMG/
virsh define ~/vm-backup/$VM.xml
virsh start $VM
```

### Kết nối và thông báo

```bash
virsh uri                                # kết nối đang dùng (qemu:///system)
virsh version
virsh nodeinfo                           # CPU/RAM của host
virsh pool-list --all                    # storage pool
virsh vol-list default                   # volume trong pool default
```

---

## Lệnh kiểm tra ổ đĩa / storage

### Trên host

```bash
lsblk                                    # cây đĩa và mount point
lsblk -o NAME,SIZE,TYPE,VENDOR,MODEL,WWN,MOUNTPOINTS
findmnt                                  # cây mount
findmnt /dev/mapper/mpathb               # không in gì = chưa mount
blkid /dev/mapper/mpathb || echo "không có filesystem"
df -h /var                               # dung lượng thư mục chứa image

# iSCSI
iscsiadm -m session                      # session đang đăng nhập
iscsiadm -m session -P3 | grep -E 'Target:|Current Portal|iSCSI Session State|Attached scsi disk'
iscsiadm -m node -o show | grep -E 'node.name|node.startup'   # startup nên là automatic

# Multipath
multipath -ll                            # tất cả map
multipath -ll mpathb
multipath -r                             # nạp lại
cat /etc/multipath/wwids

# Đối chiếu WWID (đảm bảo đúng LUN)
/lib/udev/scsi_id -g -u -d /dev/sdb      # WWID của sdb
cat /sys/block/dm-0/dm/uuid              # mpath-<WWID>
ls /sys/block/dm-0/slaves                # các sdX nằm dưới dm-0
ls -l /dev/mapper/mpathb                 # mpathb -> ../dm-N
ls -l /dev/disk/by-path | grep iscsi

# Ai đang giữ mpathb
fuser -v /dev/mapper/mpathb
for d in $(virsh list --all --name); do virsh domblklist $d | grep mpathb && echo "VM $d"; done

# Sức khỏe
dmesg -T | grep -iE 'I/O error|multipath|iscsi|blocked for more' | tail -20
iostat -x 2                              # tải đĩa theo thời gian thực
```

### File image

```bash
qemu-img info $IMG/$VM.qcow2
qemu-img info --backing-chain $IMG/$VM.qcow2
qemu-img check $IMG/$VM.qcow2
ls -lh $IMG/
```

### Trong VM

```bash
lsblk -o NAME,SIZE,TYPE,SERIAL,MOUNTPOINTS
ls -l /dev/disk/by-id/
df -h /
free -h
nproc
```

---

## Khái niệm tên đĩa

```
LUN trên storage (WWID 36001405d4868733ad614316bf13b6462)
   │  qua mạng iSCSI
   ▼
sdb            ← path, tên tạm kernel đặt (có thể đổi sau reboot)
   │  multipath gộp theo WWID
   ▼
dm-0           ← thiết bị Device Mapper (tên kernel)
 = /dev/mapper/mpathb   ← tên thân thiện, cùng thiết bị với dm-0
   │  libvirt gắn vào VM
   ▼
vdb            ← tên trong VM
```

| Tên | Ở đâu | Ý nghĩa |
|---|---|---|
| WWID | Storage đặt | "Số CCCD" của LUN, không đổi. Cùng WWID = cùng LUN vật lý |
| `sdb` | Host | Tên của một path, **không cố định**. Đừng bench trực tiếp |
| `dm-0` / `mpathb` | Host | Thiết bị multipath, VM đi qua đường này |
| `vdb` | Trong VM | Tên đĩa trong VM (`v` = virtio) |
| Mount point | Host/VM | Thư mục mà một đĩa được gắn vào. `MOUNTPOINTS` trống = chưa mount |

---

## Khi `virsh start` / `virt-install` bị treo (DBus Timeout)

Triệu chứng: `GDBus.Error:org.freedesktop.DBus.Error.Timeout` hoặc `systemd-machined: Failed to start machine scope: Connection timed out`.

```bash
systemctl is-active dbus libvirtd systemd-machined
timeout 40 systemd-run --scope --quiet true; echo "scope exit=$?"     # phải exit=0
journalctl -u libvirtd -u systemd-machined --since "5 min ago" --no-pager | tail -30
ps -eo pid,stat,wchan:25,cmd | awk '$2 ~ /^D/'                         # tiến trình kẹt I/O
dmesg -T | tail -30
uptime -s; uptime
grep VmRSS /proc/1/status                                              # systemd phình RAM bất thường?
```

Cách xử lý, từ nhẹ đến mạnh:

```bash
# 1. Restart service
systemctl restart systemd-machined libvirtd
virsh list --all

# 2. Re-exec systemd qua D-Bus (không đụng VM)
timeout 20 busctl call org.freedesktop.systemd1 /org/freedesktop/systemd1 \
  org.freedesktop.systemd1.Manager Reexecute

# 3. Nếu vẫn lỗi: tín hiệu cho PID 1 (với systemd, SIGTERM = re-exec)
kill -TERM 1; sleep 5

# 4. Cuối cùng: reboot cưỡng bức (báo người dùng lab, tắt VM trước)
sync; sync; reboot -f
```

Mẹo: luôn bọc lệnh có thể treo bằng `timeout`, và không Ctrl+C giữa lúc `virt-install` đang tạo domain.

---

## Lưu ý an toàn

- Không lưu email, mật khẩu hay token trong file ghi chú. Mật khẩu `123456` trong cloud-init chỉ dùng cho VM lab.
- Trước khi gõ `YES` ở `bench.sh`, luôn đọc lại thiết bị mà script in ra.
- Test ghi hao tuổi thọ SSD: giảm `QDS` và `runtime` nếu chạy nhiều lần.
- Bench trên host và bench trong VM đều dùng cùng một LUN: không chạy đồng thời.
