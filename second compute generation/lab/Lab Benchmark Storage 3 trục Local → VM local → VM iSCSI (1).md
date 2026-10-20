# Lab Benchmark Storage 3 trục: Local → VM local → VM iSCSI



### 1.4. Script `bench.sh` (tập trung P99 và P99.99)

Tạo file `bench.sh` ở com07 (cho trục A) và trong VM (cho trục B, C). Script có 3 lớp bảo vệ: từ chối nếu thiết bị đang mount, bắt gõ `YES`, và luôn resolve về thiết bị thật.

```bash
cat > bench.sh << 'EOF'
#!/bin/bash
# Usage: sudo [QDS="1 4 16"] [SEQ=1] ./bench.sh <device> <tag> [runtime_sec]
# VD: sudo ./bench.sh /dev/disk/by-id/wwn-0x... A_local 60
set -u
DEV=$1; TAG=$2; RT=${3:-60}
QDS=${QDS:-"1 2 4 8 16 32 64 128"}   # các mức queue depth cần đo
BS=4k                                 # block size chuẩn
REAL=$(readlink -f "$DEV")

if lsblk -nro MOUNTPOINT "$REAL" 2>/dev/null | grep -q . ; then
  echo "ABORT: $REAL (hoặc phân vùng con) đang được mount"; exit 1
fi
read -r -p "Test GHI sẽ PHÁ dữ liệu trên $REAL. Gõ YES để tiếp tục: " ANS
[ "$ANS" = "YES" ] || { echo "Huỷ."; exit 1; }

OUT=./result_$TAG; mkdir -p "$OUT"
# direct=1: bỏ qua page cache. sync=0: không ép flush từng I/O (O_SYNC tắt)
COMMON="--filename=$REAL --size=50G --direct=1 --sync=0 --ioengine=libaio \
 --time_based --runtime=$RT --ramp_time=10 --group_reporting \
 --clat_percentiles=1 --percentile_list=99:99.99 --output-format=json"

run() { n=$1; shift; echo ">>> $n"; fio --name=$n $COMMON "$@" --output=$OUT/$n.json; }

# numjobs=1 => QD = số I/O đang chờ. QD=1 là phép đo latency thuần.
for QD in $QDS; do
  run randread_qd$QD  --rw=randread  --bs=$BS --iodepth=$QD --numjobs=1
  run randwrite_qd$QD --rw=randwrite --bs=$BS --iodepth=$QD --numjobs=1
  run mixed_qd$QD     --rw=randrw --rwmixread=70 --bs=$BS --iodepth=$QD --numjobs=1
done

# Tuỳ chọn: băng thông tuần tự 1M, QD8 (bật bằng SEQ=1)
if [ "${SEQ:-0}" = "1" ]; then
  run seqread_bs1M  --rw=read  --bs=1M --iodepth=8 --numjobs=1
  run seqwrite_bs1M --rw=write --bs=1M --iodepth=8 --numjobs=1
fi

{
  echo -e "test\tparam\tdir\tIOPS\tMiB/s\tavg_ms\tP99_ms\tP99.99_ms"
  for f in $(ls $OUT/*.json | sort -V); do
    n=$(basename "$f" .json); t=${n%%_*}; p=${n#*_}
    jq -r --arg t "$t" --arg p "$p" '.jobs[0] | to_entries[]
      | select(.key=="read" or .key=="write") | select(.value.io_bytes>0)
      | [$t, $p, .key, (.value.iops|floor), ((.value.bw/1024)|floor),
         ((.value.clat_ns.mean/1e6*1000|round)/1000),
         ((.value.clat_ns.percentile["99.000000"]/1e6*1000|round)/1000),
         ((.value.clat_ns.percentile["99.990000"]/1e6*1000|round)/1000)] | @tsv' "$f"
  done
} | column -t -s $'\t' | tee "$OUT/summary.txt"
EOF
chmod +x bench.sh
```

Ghi chú về script:

- **Thông số đo:** `bs=4k`, `numjobs=1`, `direct=1`, `sync=0`, `libaio`, quét QD = 1, 2, 4, 8, 16, 32, 64, 128. Mỗi QD chạy 3 test: randread, randwrite, mixed 70/30. Vì `numjobs=1` nên QD chính là số I/O đang chờ; lần đo cũ (4 jobs × QD32) tương đương QD128 ở đây.
- **QD=1 là phép đo latency thuần:** mỗi lúc chỉ một I/O, không xếp hàng. `direct=1` bỏ qua page cache của OS. `sync=0` không ép flush từng I/O, nên số ghi phụ thuộc cache ghi của đĩa hoặc backstore. Muốn đo ghi bền vững thì chạy riêng một lượt với `--sync=1`.
- Dùng **`clat`** (completion latency) với `--percentile_list=99:99.99`, đơn vị trong bảng là **ms**; thêm cột `avg_ms` để đối chiếu với P99.
- **Thời gian:** mặc định 60s + 10s ramp, 24 lượt ≈ 28 phút mỗi trục. Rút gọn bằng `sudo QDS="1 4 16 64" ./bench.sh ...`. Muốn thêm băng thông tuần tự 1M QD8 thì đặt `SEQ=1`.
- **P99.99 cần nhiều mẫu:** số mẫu ≈ IOPS × runtime. Test ghi chậm ở QD thấp có ít mẫu nên P99.99 chỉ để tham khảo; cần chắc thì tăng runtime (tham số thứ 3, ví dụ `300`).
- Nếu cột P99 hoặc P99.99 ra `null`, xem tên khoá thật bằng: `jq '.jobs[0].read.clat_ns.percentile' result_*/randread_qd1.json`.

---

## 2. Trục A: Local trên com07 → disk

*Chạy trên **com07**. Không có VM hay target nào đang dùng `sda`.*

```bash
sudo fuser -v "$SDA"          # phải không in tiến trình nào
sudo ./bench.sh "$SDA" A_local 60
```

Kết quả nằm ở `./result_A_local/` (`summary.txt` là bảng tóm tắt). Chạy lại lần 2 với tag `A_local_2`.

-  Vẫn bật cache DRAM của ssd : --sync 0 
```
root@vhhl1c2lab2com07:~/result_A_local# cat summary.txt
test       param  dir    IOPS   MiB/s  avg_ms  P99_ms  P99.99_ms
mixed      qd1    read   10747  41     0.076   0.253   0.481
mixed      qd1    write  4609   18     0.03    0.112   0.142
mixed      qd2    read   17144  66     0.093   0.198   0.403
mixed      qd2    write  7349   28     0.045   0.134   0.218
mixed      qd4    read   23856  93     0.134   0.338   0.569
mixed      qd4    write  10230  39     0.068   0.159   0.253
mixed      qd8    read   35989  140    0.176   0.399   0.659
mixed      qd8    write  15421  60     0.099   0.241   0.301
mixed      qd16   read   44916  175    0.279   0.545   0.913
mixed      qd16   write  19245  75     0.171   0.326   0.412
mixed      qd32   read   51923  202    0.481   0.872   1.368
mixed      qd32   write  22257  86     0.307   0.553   0.725
mixed      qd64   read   56525  220    0.749   1.63    2.867
mixed      qd64   write  24229  94     0.881   2.736   5.014
mixed      qd128  read   51748  202    1.757   5.014   13.435
mixed      qd128  write  22184  86     1.659   6.128   15.27
randread   qd1    read   10441  40     0.093   0.159   0.214
randread   qd2    read   21813  85     0.088   0.179   0.338
randread   qd4    read   31043  121    0.126   0.224   0.33
randread   qd8    read   48436  189    0.162   0.281   0.424
randread   qd16   read   64230  250    0.246   0.436   0.586
randread   qd32   read   83246  325    0.382   0.586   0.766
randread   qd64   read   87585  342    0.727   1.679   3.26
randread   qd128  read   86603  338    1.474   3.883   11.469
randwrite  qd1    write  35426  138    0.026   0.029   0.116
randwrite  qd2    write  28478  111    0.068   0.073   0.131
randwrite  qd4    write  55251  215    0.07    0.142   0.165
randwrite  qd8    write  30286  118    0.262   0.285   0.465
randwrite  qd16   write  29718  116    0.535   0.709   0.807
randwrite  qd32   write  30200  117    1.056   1.253   2.023
randwrite  qd64   write  29393  114    2.173   5.145   10.158
randwrite  qd128  write  57006  222    2.242   6.98    26.87
root@vhhl1c2lab2com07:~/result_A_local#

```
---

## 3. Trục B: com8 ->  com07 → disk


```

cat > bench.sh << 'EOF'
#!/bin/bash
# Usage: sudo [QDS="1 4 16"] [SEQ=1] ./bench.sh <device> <tag> [runtime_sec]
# VD: sudo ./bench.sh /dev/mapper/mpathb B_iscsi 60
set -u

DEV=$1; TAG=$2; RT=${3:-60}

QDS=${QDS:-"1 2 4 8 16 32 64 128"}   # các mức queue depth cần đo
BS=4k                                 # block size chuẩn
REAL=$(readlink -f "$DEV")

if lsblk -nro MOUNTPOINT "$REAL" 2>/dev/null | grep -q . ; then
  echo "ABORT: $REAL (hoặc phân vùng con) đang được mount"; exit 1
fi

read -r -p "Test GHI sẽ PHÁ dữ liệu trên $REAL. Gõ YES để tiếp tục: " ANS
[ "$ANS" = "YES" ] || { echo "Huỷ."; exit 1; }

OUT=./result_$TAG; mkdir -p "$OUT"

# direct=1: bỏ qua page cache. sync=0: không ép flush từng I/O (O_SYNC tắt). ioengine=libaio: tương đương io=native
COMMON="--filename=$REAL --size=50G --direct=1 --sync=0 --ioengine=libaio \
 --time_based --runtime=$RT --ramp_time=10 --group_reporting \
 --clat_percentiles=1 --percentile_list=99:99.99 --output-format=json"

run() { n=$1; shift; echo ">>> $n"; fio --name=$n $COMMON "$@" --output=$OUT/$n.json; }

# numjobs=1 => QD = số I/O đang chờ. QD=1 là phép đo latency thuần.
for QD in $QDS; do
  run randread_qd$QD  --rw=randread  --bs=$BS --iodepth=$QD --numjobs=1
  run randwrite_qd$QD --rw=randwrite --bs=$BS --iodepth=$QD --numjobs=1
  run mixed_qd$QD     --rw=randrw --rwmixread=70 --bs=$BS --iodepth=$QD --numjobs=1
done

# Tuỳ chọn: băng thông tuần tự 1M, QD8 (bật bằng SEQ=1)
if [ "${SEQ:-0}" = "1" ]; then
  run seqread_bs1M  --rw=read  --bs=1M --iodepth=8 --numjobs=1
  run seqwrite_bs1M --rw=write --bs=1M --iodepth=8 --numjobs=1
fi

{
echo -e "test\tparam\tdir\tIOPS\tMiB/s\tavg_ms\tP99_ms\tP99.99_ms"

for f in $(ls $OUT/*.json | sort -V); do
    n=$(basename "$f" .json); t=${n%%_*}; p=${n#*_}
    jq -r --arg t "$t" --arg p "$p" '.jobs[0] | to_entries[] \
      | select(.key=="read" or .key=="write") | select(.value.io_bytes>0) \
      | [$t, $p, .key, (.value.iops|floor), ((.value.bw/1024)|floor), \
         (((.value.clat_ns.mean // 0)/1e6*1000|round)/1000), \
         (((.value.clat_ns.percentile["99.000000"] // 0)/1e6*1000|round)/1000), \
         (((.value.clat_ns.percentile["99.990000"] // 0)/1e6*1000|round)/1000)] | @tsv' "$f"
done
} | column -t -s $'\t' | tee "$OUT/summary.txt"

EOF

chmod +x bench.sh
```

```
sudo SEQ=1 ./bench.sh /dev/mapper/mpathb B_iscsi_mpathb 60
```

```
root@vhhl1lab2com09:~# OUT="./result_B_iscsi_mpathb"

{
echo -e "test\tparam\tdir\tIOPS\tMiB/s\tavg_ms\tP99_ms\tP99.99_ms"

for f in $(ls $OUT/*.json | sort -V); do
    n=$(basename "$f" .json); t=${n%%_*}; p=${n#*_}
    jq -r --arg t "$t" --arg p "$p" '
      .jobs[0] | to_entries[]
      | select(.key == "read" or .key == "write")
      | select(.value.io_bytes > 0)
      | [
          $t,
          $p,
          .key,
          (.value.iops | floor),
          ((.value.bw / 1024) | floor),
          ((((.value.clat_ns.mean // 0) / 1e6) * 1000 | round) / 1000),
          ((((.value.clat_ns.percentile["99.000000"] // 0) / 1e6) * 1000 | round) / 1000),
          ((((.value.clat_ns.percentile["99.990000"] // 0) / 1e6) * 1000 | round) / 1000)
        ] | @tsv
    ' "$f"
done
} | column -t -s $'\t' | tee "$OUT/summary.txt"
test       param  dir    IOPS   MiB/s  avg_ms  P99_ms  P99.99_ms
mixed      qd1    read   1854   7      0.374   0.7     0.807
mixed      qd1    write  793    3      0.316   0.643   0.7
mixed      qd2    read   4451   17     0.319   0.578   0.782
mixed      qd2    write  1911   7      0.266   0.545   0.725
mixed      qd4    read   11140  43     0.265   0.569   0.823
mixed      qd4    write  4777   18     0.198   0.498   0.7
mixed      qd8    read   25547  99     0.242   0.506   0.831
mixed      qd8    write  10948  42     0.152   0.379   0.586
mixed      qd16   read   42286  165    0.294   0.545   0.897
mixed      qd16   write  18117  70     0.184   0.309   0.489
mixed      qd32   read   51408  200    0.488   0.889   1.335
mixed      qd32   write  22033  86     0.302   0.457   0.676
mixed      qd64   read   50392  196    0.87    1.876   4.293
mixed      qd64   write  21597  84     0.922   2.376   4.489
mixed      qd128  read   51571  201    1.721   4.555   9.896
mixed      qd128  write  22104  86     1.763   4.882   8.978
randread   qd1    read   2048   8      0.46    0.758   0.823
randread   qd2    read   6101   23     0.317   0.545   0.733
randread   qd4    read   14521  56     0.269   0.553   0.75
randread   qd8    read   36608  143    0.215   0.403   0.651
randread   qd16   read   59767  233    0.264   0.395   0.586
randread   qd32   read   74770  292    0.424   0.586   0.848
randread   qd64   read   76096  297    0.838   1.909   3.49
randread   qd128  read   75886  296    1.684   4.358   8.716
randwrite  qd1    write  3727   14     0.256   0.578   0.618
randwrite  qd2    write  12225  47     0.158   0.444   0.651
randwrite  qd4    write  25825  100    0.151   0.358   0.561
randwrite  qd8    write  29676  115    0.266   0.309   0.514
randwrite  qd16   write  29845  116    0.532   0.676   0.963
randwrite  qd32   write  29559  115    1.078   1.303   1.63
randwrite  qd64   write  30966  120    2.064   4.817   9.241
randwrite  qd128  write  30746  120    4.16    10.945  32.375
seqread    bs1M   read   541    541    14.727  15.27   19.792
seqwrite   bs1M   write  303    303    26.264  34.865  47.972

```
## 4. Trục C: VM trên com08 → iSCSI → disk của com07

### 4.1. com07: export `sda` qua iSCSI (block backstore)

*Chạy trên **com07**, sau khi đã nhả `sda` ở bước 3.4.*

```bash
sudo targetcli
```

	Trong `targetcli` (thay `<WWN_PATH>` bằng đúng giá trị `$SDA`, in ra bằng `echo $SDA` ở shell khác):

```text
# 1. Backstore block trỏ thẳng vào ổ cứng vật lý thô
cd /backstores/block
create disk1 <WWN_PATH>

# 2. Tạo Target iSCSI
cd /iscsi
create iqn.2026-09.local.fptcloud:storage.target1

# 3. Gắn đĩa vào LUN 0
cd /iscsi/iqn.2026-09.local.fptcloud:storage.target1/tpg1/luns
create /backstores/block/disk1

# 4. MỞ QUYỀN TRUY CẬP (Bắt buộc để com07 login được)
cd /iscsi/iqn.2026-09.local.fptcloud:storage.target1/tpg1
set attribute generate_node_acls=1 demo_mode_write_protect=0

# 5. CẤU HÌNH CACHE CHO BENCHMARK
# Xem giá trị hiện tại (1: bật write-back giả lập, 0: tắt)
cd /backstores/block/disk1
get attribute emulate_write_cache

# Nếu muốn tắt sạch cache LIO, ép ghi thẳng xuống SSD:
# set attribute emulate_write_cache=0

# 6. Kiểm tra lại toàn bộ cây cấu hình
cd /
ls

# 7. Lưu cấu hình và thoát
saveconfig
exit
```

Ghi lại giá trị `emulate_write_cache` (0 = write-through, 1 = write-back) vào kết quả vì nó ảnh hưởng trực tiếp đến latency ghi. Nếu `ufw` đang bật thì mở cổng `3260` và `5201` cho `10.11.4.24`.


### 4.4. Network baseline (com08 → com07)

Đo mốc mạng để biết trần của đường truyền trước khi đo storage.

Trên **com07**:

```bash
iperf3 -s
```

Trên **com08**:

```bash
ping -c 20 10.11.4.23
iperf3 -c 10.11.4.23 -t 30 -i 5          # 1 luồng
iperf3 -c 10.11.4.23 -t 30 -P 4          # 4 luồng
```


### 4.5. com08: login iSCSI

```bash
sudo iscsiadm -m discovery -t sendtargets -p 10.11.4.23:3260

sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.23:3260 --login

sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.23:3260 \
  -o update -n node.startup -v automatic

lsblk
sudo iscsiadm -m session -P 3 | grep -E "Attached scsi disk|Current Portal"

root@vhhl1lab2com08:~# export WWID=$(sudo /lib/udev/scsi_id -g -u -d /dev/sdb)
echo $WWID        # phải KHÁC 360014050ebbd125c1fb459f8e0c917d4

sudo multipath -a $WWID
sudo systemctl restart multipathd
sudo multipath -r
sudo multipath -ll
ls -l /dev/mapper/

root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo iscsiadm -m discovery -t sendtargets -p 10.11.4.23:3260
10.11.4.23:3260,1 iqn.2026-09.local.fptcloud:storage.target1
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.23:3260 --
Logging in to [iface: default, target: iqn.2026-09.local.fptcloud:storage.target1, portal: 10.11.4.23,3260]
Login to [iface: default, target: iqn.2026-09.local.fptcloud:storage.target1, portal: 10.11.4.23,3260] successful.
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.23:3260 \
  -o update -n node.startup -v automatic
root@vhhl1c2lab2com09:/var/lib/libvirt/images# lsblk
NAME   MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0    7:0    0  63.9M  1 loop /snap/core20/2318
loop1    7:1    0    87M  1 loop /snap/lxd/29351
loop3    7:3    0    74M  1 loop /snap/core22/2955
loop4    7:4    0 115.2M  1 loop /snap/lxd/40911
loop5    7:5    0  50.3M  1 loop /snap/snapd/27738
loop6    7:6    0  63.8M  1 loop /snap/core20/2922
loop7    7:7    0  44.7M  1 loop /snap/snapd/28254
sda      8:0    0 446.6G  0 disk
├─sda1   8:1    0     1G  0 part /boot/efi
├─sda2   8:2    0     3G  0 part /boot
├─sda3   8:3    0    20G  0 part /home
├─sda4   8:4    0    50G  0 part /var/log
└─sda5   8:5    0 372.6G  0 part /
sdb      8:16   0 447.1G  0 disk
root@vhhl1c2lab2com09:/var/lib/libvirt/images# export WWID=$(sudo /lib/udev/scsi_id -g -u -d /dev/sdb)
root@vhhl1c2lab2com09:/var/lib/libvirt/images# echo $WWID
36001405d4868733ad614316bf13b6462
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo multipath -a $WWID
wwid '36001405d4868733ad614316bf13b6462' added
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo systemctl restart multipathd
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo multipath -r
root@vhhl1c2lab2com09:/var/lib/libvirt/images# sudo multipath -ll
mpathb (36001405d4868733ad614316bf13b6462) dm-0 LIO-ORG,disk1
size=447G features='0' hwhandler='1 alua' wp=rw
`-+- policy='service-time 0' prio=50 status=active
  `- 1:0:0:0 sdb 8:16 active ready running
root@vhhl1c2lab2com09:/var/lib/libvirt/images# ls -l /dev/mapper/
total 0
crw------- 1 root root 10, 236 Sep 29 20:45 control
lrwxrwxrwx 1 root root       7 Oct 18 08:38 mpathb -> ../dm-0
root@vhhl1c2lab2com09:/var/lib/libvirt/images#


```

Xác nhận có ổ 447G mới (tên `sdX` do hệ thống đặt, gọi tạm là `sdX`). **Lưu ý: đây là LUN mới nên WWID mới, không dùng lại WWID của lab cũ.**

### 4.6. com08: multipath

```bash
# Lấy WWID của LUN mới
export WWID=$(sudo /lib/udev/scsi_id -g -u -d /dev/sdX)
echo $WWID

sudo multipath -a $WWID

sudo tee /etc/multipath.conf << EOF
defaults {
    user_friendly_names yes
    find_multipaths yes
}

blacklist {
    devnode "^(loop|ram|sr)[0-9]*"
}

blacklist_exceptions {
    wwid "$WWID"
}
EOF

sudo systemctl restart multipathd
sudo multipath -r
sudo multipath -ll
ls -l /dev/mapper/
```

Kết quả mong đợi: `mpatha` (447G) bọc `sdX`. Nếu ổ OS của com08 cũng bị map, thêm WWID của nó vào `blacklist` (`wwid "<WWID_OS>"`) rồi `multipath -r` lại.

### 4.7. com08: tạo VM `vm-bench-iscsi` (cùng cấu hình với trục B)

```
test       param  dir    IOPS   MiB/s  avg_ms   P99_ms    P99.99_ms
mixed      qd1    read   1362   5      0.527    1.057     1.237
mixed      qd1    write  582    2      0.439    0.872     1.073
mixed      qd2    read   4024   15     0.361    0.692     1.057
mixed      qd2    write  1729   6      0.301    0.627     0.856
mixed      qd4    read   9047   35     0.323    0.668     0.963
mixed      qd4    write  3885   15     0.265    0.618     0.889
mixed      qd8    read   23562  92     0.258    0.514     0.881
mixed      qd8    write  10104  39     0.179    0.424     0.791
mixed      qd16   read   40591  158    0.302    0.545     0.881
mixed      qd16   write  17384  67     0.205    0.33      0.668
mixed      qd32   read   52051  203    0.467    0.774     1.352
mixed      qd32   write  22312  87     0.336    0.473     0.815
mixed      qd64   read   48738  190    0.905    1.663     2.802
mixed      qd64   write  20892  81     0.942    1.761     2.933
mixed      qd128  read   51027  199    1.714    3.391     4.751
mixed      qd128  write  21869  85     1.844    3.457     4.751
randread   qd1    read   1267   4      0.762    1.106     1.81
randread   qd2    read   6251   24     0.314    0.635     0.913
randread   qd4    read   11522  45     0.343    0.7       0.987
randread   qd8    read   31779  124    0.248    0.537     0.864
randread   qd16   read   55376  216    0.286    0.44      0.889
randread   qd32   read   72846  284    0.437    0.61      1.073
randread   qd64   read   73550  287    0.868    1.45      2.474
randread   qd128  read   72535  283    1.762    3.391     5.079
randwrite  qd1    write  2233   8      0.441    0.75      0.872
randwrite  qd2    write  10203  39     0.191    0.553     0.758
randwrite  qd4    write  19162  74     0.205    0.522     0.799
randwrite  qd8    write  30126  117    0.261    0.313     0.668
randwrite  qd16   write  30780  120    0.516    0.651     0.913
randwrite  qd32   write  30057  117    1.061    1.204     2.007
randwrite  qd64   write  30521  119    2.094    4.686     9.11
randwrite  qd128  write  30021  117    4.26     11.731    31.85
seqread    bs1M   read   16     16     489.069  2055.209  3271.557
seqwrite   bs1M   write  356    356    22.381   31.326    40.632

```

```bash
command -v virsh virt-install qemu-img cloud-localds || echo "thieu goi, chay: sudo apt install -y libvirt-clients virtinst qemu-utils cloud-image-utils"
virsh net-list | grep default
ls -lh /var/lib/libvirt/images/noble-base.img
```

N

```bash
export IMG=/var/lib/libvirt/images
export VM=vm1
export BASE=$IMG/noble-base.img
export PROXY=http://10.10.2.22:3001

# Host tải gói/image qua proxy (chỉ cần khi phải wget)
export HTTP_PROXY=$PROXY HTTPS_PROXY=$PROXY http_proxy=$PROXY https_proxy=$PROXY

[ -f $BASE ] || { echo "Thieu $BASE"; exit 1; }
sudo cp $BASE $IMG/$VM.qcow2
sudo qemu-img resize $IMG/$VM.qcow2 +10G
```

**Bước 2: Cloud-init (proxy cho apt, biến proxy, tool benchmark)**

`EOF` không có nháy đơn để `$VM` và `$PROXY` được thay giá trị.

```bash
cat > /tmp/$VM-user-data << EOF
#cloud-config
hostname: $VM
password: 123456
chpasswd: { expire: false }
ssh_pwauth: true

# Proxy cho apt (apply truoc khi cai goi)
apt:
  http_proxy: $PROXY
  https_proxy: $PROXY

# Proxy cho shell/ma khac (login moi deu co)
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
grep -A2 '^apt:' /tmp/$VM-user-data      # kiem tra proxy da duoc thay gia tri
```

**Bước 3: Tạo VM**

```bash
sudo virt-install \
  --name $VM --ram 2048 --vcpus 2 \
  --disk path=$IMG/$VM.qcow2,format=qcow2,bus=virtio \
  --disk path=$IMG/$VM-seed.img,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics none --import --noautoconsole
```

**Bước 4: Vào VM và kiểm tra** (đợi khoảng 1-2 phút để cloud-init cài gói)

```bash
sleep 60
IP=$(virsh domifaddr $VM | awk '/ipv4/ {print $4}' | cut -d/ -f1); echo $IP
ssh ubuntu@$IP            # mật khẩu 123456
```

Trong VM:

```bash
cloud-init status --wait          # phai ra "status: done"
command -v fio jq column iostat   # du tool
env | grep -i proxy               # co 4 bien proxy
cat /etc/apt/apt.conf.d/99proxy
```

Nếu `apt` không tải được gói thì xem `grep -i proxy /var/log/cloud-init.log` và kiểm tra VM có tới được `10.10.2.22:3001` không (`curl -I -x $http_proxy http://archive.ubuntu.com`).

**Hàm `wait_off`**: script cũ của bạn dùng `wait_off` nhưng chưa định nghĩa. Thêm vào trước khi dùng phần clone/backup:

```bash
wait_off() { while [ "$(virsh domstate "$1")" != "shut off" ]; do sleep 2; done; }
```

**Lưu ý**

- Biến proxy chỉ có hiệu lực ở phiên đăng nhập mới. Phiên SSH đang mở thì chạy `source /etc/environment` hoặc đăng nhập lại.
- Muốn đổi cấu hình cloud-init sau khi VM đã chạy lần đầu, phải tạo VM mới (hoặc `sudo cloud-init clean` trong VM rồi reboot), vì cloud-init chỉ chạy một lần.
- Ghi chú bạn dán có chứa **email và mật khẩu ở dạng văn bản thường**. Hãy đổi mật khẩu đó và đừng lưu trong file ghi chú hay dán vào chat.



---

## 5. Bảng kết quả

Điền từ `summary.txt` của từng trục (đơn vị latency: ms). Mỗi ô: `IOPS hoặc MiB/s | P99 | P99.99`.

### 5.1. Network baseline (com08 → com07)

| Chỉ số | Giá trị |
| --- | --- |
| Ping avg |  |
| iperf3 1 luồng |  |
| iperf3 4 luồng |  |
| `emulate_write_cache` của backstore |  |

### 5.2. fio

Mỗi ô ghi `IOPS | P99 | P99.99` (latency tính bằng ms). Thông số chung: `bs=4k`, `direct=1`, `sync=0`, `numjobs=1`. QD1 là latency thuần.

**Rand read 4K**

| QD | Trục A: local | Trục B: VM local | Trục C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 |  |  |  |
| 2 |  |  |  |
| 4 |  |  |  |
| 8 |  |  |  |
| 16 |  |  |  |
| 32 |  |  |  |
| 64 |  |  |  |
| 128 |  |  |  |

**Rand write 4K**

| QD | Trục A: local | Trục B: VM local | Trục C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 |  |  |  |
| 2 |  |  |  |
| 4 |  |  |  |
| 8 |  |  |  |
| 16 |  |  |  |
| 32 |  |  |  |
| 64 |  |  |  |
| 128 |  |  |  |

**Mixed 70/30 4K – read**

| QD | Trục A: local | Trục B: VM local | Trục C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 |  |  |  |
| 2 |  |  |  |
| 4 |  |  |  |
| 8 |  |  |  |
| 16 |  |  |  |
| 32 |  |  |  |
| 64 |  |  |  |
| 128 |  |  |  |

**Mixed 70/30 4K – write**

| QD | Trục A: local | Trục B: VM local | Trục C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 |  |  |  |
| 2 |  |  |  |
| 4 |  |  |  |
| 8 |  |  |  |
| 16 |  |  |  |
| 32 |  |  |  |
| 64 |  |  |  |
| 128 |  |  |  |

**Tuỳ chọn (`SEQ=1`): băng thông tuần tự 1M QD8**, mỗi ô ghi `MiB/s | P99 | P99.99`

| Test | Trục A: local | Trục B: VM local | Trục C: VM qua iSCSI |
| --- | --- | --- | --- |
| Seq read |  |  |  |
| Seq write |  |  |  |

### 5.3. Cách đọc P99 và P99.99

- **QD1** (`direct=1`, `sync=0`) cho latency thuần: không có hàng đợi, nên P99 và P99.99 ở QD1 là mức tốt nhất mà đường đi đó đạt được. Tăng QD thì IOPS tăng tới một điểm bão hoà, sau đó chỉ còn latency tăng thêm. Điểm gãy đó là mức tải hợp lý của từng trục.
- **P99** là mốc mà 99% I/O hoàn thành nhanh hơn, phản ánh trải nghiệm "bình thường xấu". **P99.99** chỉ 1 trong 10.000 I/O chậm hơn mốc này, phản ánh đuôi latency (GC của SSD, flush, nghẽn hàng đợi, retransmit).
- So P99.99 giữa các trục: nếu **A→B** tăng đột biến thì nghi virtio/QEMU, nếu **B→C** tăng đột biến thì nghi mạng, iSCSI hoặc multipath.
- Băng thông seqread ở trục C (chạy với `SEQ=1`) có thể bị chặn bởi băng thông mạng. So MiB/s với kết quả iperf3 4 luồng.
- Đừng kết luận nguyên nhân khi chưa đo thêm. Các nhận định về cache hay write-back chỉ là giả thuyết cho đến khi đổi cấu hình và đo lại.

---

## Phụ lục: lỗi thường gặp

| Triệu chứng                            | Nguyên nhân / cách xử lý                                                                             |
| -------------------------------------- | ---------------------------------------------------------------------------------------------------- |
| `apt` / `wget` không tải được          | Thiếu proxy: `env \| grep -i proxy`, cấu hình proxy cho `apt` trên com07, com08 và VM                |
| `virsh attach-device` báo thiết bị bận | `sda` còn đang bị consumer khác dùng. Kiểm tra `sudo fuser -v "$SDA"` và LIO export (`targetcli ls`) |
| `/dev/mapper/mpatha` không xuất hiện   | Chưa `multipath -a` với WWID **mới**; hoặc blacklist nhầm. Chạy lại `multipath -r`                   |
| Login iSCSI bị từ chối                 | Chưa tạo ACL với đúng InitiatorName của **com08** trên com07                                         |
| `iperf3` không kết nối được            | Chưa chạy `iperf3 -s` trên com07, hoặc firewall chặn cổng 5201                                       |
| `bench.sh` báo ABORT mount             | Thiết bị có mountpoint, kiểm tra lại đúng đĩa trước khi làm tiếp                                     |
| Bảng P99 / P99.99 ra `null`            | Sai tên khoá percentile trong JSON, xem lệnh `jq` ở mục 1.4                                          |
| Sau reboot đổi tên `sda` ↔ `sdb`       | Luôn dùng `/dev/disk/by-id/wwn-...`, không dùng `/dev/sda` trực tiếp                                 |
