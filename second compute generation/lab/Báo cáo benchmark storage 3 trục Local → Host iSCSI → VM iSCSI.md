# Báo cáo benchmark storage 3 trục: Local → Host iSCSI → VM iSCSI

Ngày: 18/10/2026 · Công cụ: fio 3.36 · Script: `bench.sh`

## 1. Tóm tắt

Đo cùng một ổ SSD (`sda` của com07) qua ba đường đi để biết mỗi lớp (iSCSI, ảo hóa) làm tăng latency và giảm IOPS bao nhiêu.

- **Latency ở QD thấp tăng mạnh khi đi qua iSCSI.** Rand read 4K ở QD1: avg 0,093 ms (local) → 0,46 ms (host qua iSCSI) → 0,762 ms (VM qua iSCSI). IOPS giảm từ 10.441 xuống 2.048 và 1.267. Rand write QD1: 35.426 → 3.727 → 2.233 IOPS.
- **Mạng iSCSI cộng khoảng 0,23–0,37 ms mỗi I/O, VM cộng thêm khoảng 0,19–0,30 ms.** Đây là mức chênh latency trung bình ở QD1 (mục 4.1).
- **Ở QD cao, ba trục gần như bằng nhau.** Rand read đỉnh ở QD64: 87.585 (local), 76.096 (host), 73.550 (VM) IOPS, tức host đạt 87% và VM đạt 84% so với local. Mixed 70/30 ở QD32 trở lên hội tụ khoảng 51–52 nghìn IOPS đọc ở cả ba trục.
- **Rand write ở host và VM bão hòa ổn định khoảng 30.000 IOPS (≈ 117–120 MiB/s) từ QD8 đến QD128**, và local cũng nằm quanh mức này ở QD8–QD64. Vì vậy đây nhiều khả năng là giới hạn ở phía ổ hoặc target chứ không phải do mạng hay VM (giả thuyết, cần đo xác nhận, xem mục 5).
- **Có hai kết quả bất thường cần đo lại:** (1) rand write local nhảy lên 55–57 nghìn IOPS ở QD4 và QD128 xen giữa các mức \~30 nghìn; (2) seq read 1M của VM chỉ đạt 16 MiB/s (avg 489 ms) trong khi host đạt 541 MiB/s.

## 2. Mô hình và phương pháp

| Trục | Đường đi | Nơi chạy |
| --- | --- | --- |
| A | Đĩa local `sda` của com07 | com07 |
| B | Initiator → iSCSI (LIO) → `sda` của com07, qua `/dev/mapper/mpathb` | host initiator |
| C | VM (`vdb`, virtio, `cache=none`, `io=native`) → iSCSI → `sda` của com07 | VM trên com08 |

Cả ba trục đo cùng một ổ vật lý. Target: LIO block backstore `disk1` (447G), `iqn.2026-09.local.fptcloud:storage.target1`, portal `10.11.4.23:3260`. WWID LUN: `36001405d4868733ad614316bf13b6462`.

Thông số fio chung: `bs=4k`, `numjobs=1` (nên QD = `iodepth`), `direct=1`, `sync=0`, `ioengine=libaio`, `--size=50G`, `runtime=60s` cộng `ramp_time=10s`, quét QD = 1, 2, 4, 8, 16, 32, 64, 128 với ba kiểu: randread, randwrite, mixed 70% đọc/30% ghi. Seq 1M QD8 bật bằng `SEQ=1`. Chỉ số: IOPS, băng thông, latency trung bình, P99, P99.99 (clat, đơn vị ms).

Vì `sync=0`, số liệu ghi phản ánh latency tới cache ghi của đĩa hoặc backstore, không phải ghi bền vững.

## 3. Kết quả

Mỗi ô: **IOPS / P99 (ms) / P99.99 (ms)**.

### 3.1. Rand read 4K

| QD | A: local | B: host qua iSCSI | C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 | 10441 / 0.159 / 0.214 | 2048 / 0.758 / 0.823 | 1267 / 1.106 / 1.81 |
| 2 | 21813 / 0.179 / 0.338 | 6101 / 0.545 / 0.733 | 6251 / 0.635 / 0.913 |
| 4 | 31043 / 0.224 / 0.33 | 14521 / 0.553 / 0.75 | 11522 / 0.7 / 0.987 |
| 8 | 48436 / 0.281 / 0.424 | 36608 / 0.403 / 0.651 | 31779 / 0.537 / 0.864 |
| 16 | 64230 / 0.436 / 0.586 | 59767 / 0.395 / 0.586 | 55376 / 0.44 / 0.889 |
| 32 | 83246 / 0.586 / 0.766 | 74770 / 0.586 / 0.848 | 72846 / 0.61 / 1.073 |
| 64 | 87585 / 1.679 / 3.26 | 76096 / 1.909 / 3.49 | 73550 / 1.45 / 2.474 |
| 128 | 86603 / 3.883 / 11.469 | 75886 / 4.358 / 8.716 | 72535 / 3.391 / 5.079 |

### 3.2. Rand write 4K

| QD | A: local | B: host qua iSCSI | C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 | 35426 / 0.029 / 0.116 | 3727 / 0.578 / 0.618 | 2233 / 0.75 / 0.872 |
| 2 | 28478 / 0.073 / 0.131 | 12225 / 0.444 / 0.651 | 10203 / 0.553 / 0.758 |
| 4 | 55251 / 0.142 / 0.165 | 25825 / 0.358 / 0.561 | 19162 / 0.522 / 0.799 |
| 8 | 30286 / 0.285 / 0.465 | 29676 / 0.309 / 0.514 | 30126 / 0.313 / 0.668 |
| 16 | 29718 / 0.709 / 0.807 | 29845 / 0.676 / 0.963 | 30780 / 0.651 / 0.913 |
| 32 | 30200 / 1.253 / 2.023 | 29559 / 1.303 / 1.63 | 30057 / 1.204 / 2.007 |
| 64 | 29393 / 5.145 / 10.158 | 30966 / 4.817 / 9.241 | 30521 / 4.686 / 9.11 |
| 128 | 57006 / 6.98 / 26.87 | 30746 / 10.945 / 32.375 | 30021 / 11.731 / 31.85 |

### 3.3. Mixed 70/30 4K: phần đọc

| QD | A: local | B: host qua iSCSI | C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 | 10747 / 0.253 / 0.481 | 1854 / 0.7 / 0.807 | 1362 / 1.057 / 1.237 |
| 2 | 17144 / 0.198 / 0.403 | 4451 / 0.578 / 0.782 | 4024 / 0.692 / 1.057 |
| 4 | 23856 / 0.338 / 0.569 | 11140 / 0.569 / 0.823 | 9047 / 0.668 / 0.963 |
| 8 | 35989 / 0.399 / 0.659 | 25547 / 0.506 / 0.831 | 23562 / 0.514 / 0.881 |
| 16 | 44916 / 0.545 / 0.913 | 42286 / 0.545 / 0.897 | 40591 / 0.545 / 0.881 |
| 32 | 51923 / 0.872 / 1.368 | 51408 / 0.889 / 1.335 | 52051 / 0.774 / 1.352 |
| 64 | 56525 / 1.63 / 2.867 | 50392 / 1.876 / 4.293 | 48738 / 1.663 / 2.802 |
| 128 | 51748 / 5.014 / 13.435 | 51571 / 4.555 / 9.896 | 51027 / 3.391 / 4.751 |

### 3.4. Mixed 70/30 4K: phần ghi

| QD | A: local | B: host qua iSCSI | C: VM qua iSCSI |
| --- | --- | --- | --- |
| 1 | 4609 / 0.112 / 0.142 | 793 / 0.643 / 0.7 | 582 / 0.872 / 1.073 |
| 2 | 7349 / 0.134 / 0.218 | 1911 / 0.545 / 0.725 | 1729 / 0.627 / 0.856 |
| 4 | 10230 / 0.159 / 0.253 | 4777 / 0.498 / 0.7 | 3885 / 0.618 / 0.889 |
| 8 | 15421 / 0.241 / 0.301 | 10948 / 0.379 / 0.586 | 10104 / 0.424 / 0.791 |
| 16 | 19245 / 0.326 / 0.412 | 18117 / 0.309 / 0.489 | 17384 / 0.33 / 0.668 |
| 32 | 22257 / 0.553 / 0.725 | 22033 / 0.457 / 0.676 | 22312 / 0.473 / 0.815 |
| 64 | 24229 / 2.736 / 5.014 | 21597 / 2.376 / 4.489 | 20892 / 1.761 / 2.933 |
| 128 | 22184 / 6.128 / 15.27 | 22104 / 4.882 / 8.978 | 21869 / 3.457 / 4.751 |

### 3.5. Tuần tự 1M, QD8 (MiB/s / P99 ms / P99.99 ms)

| Test | A: local | B: host qua iSCSI | C: VM qua iSCSI |
| --- | --- | --- | --- |
| Seq read | chưa đo | 541 / 15.27 / 19.792 | 16 / 2055.209 / 3271.557 |
| Seq write | chưa đo | 303 / 34.865 / 47.972 | 356 / 31.326 / 40.632 |

## 4. Phân tích

### 4.1. Latency thuần (QD1)

| Phép đo QD1 | A: local | B: host | C: VM | B − A | C − B |
| --- | --- | --- | --- | --- | --- |
| Rand read avg (ms) | 0.093 | 0.46 | 0.762 | +0.367 | +0.302 |
| Rand write avg (ms) | 0.026 | 0.256 | 0.441 | +0.23 | +0.185 |
| Mixed read avg (ms) | 0.076 | 0.374 | 0.527 | +0.298 | +0.153 |

Ở QD1 mỗi I/O đi tuần tự, nên latency cộng dồn trực tiếp: đường iSCSI (mạng, LIO, multipath) thêm khoảng 0,23–0,37 ms, lớp VM (virtio, QEMU) thêm khoảng 0,15–0,30 ms. IOPS QD1 giảm tương ứng: rand read còn 20% (host) và 12% (VM) so với local. Rand write local ở QD1 rất nhanh (0,026 ms) do ghi vào cache của đĩa (`sync=0`), nên mức chênh khi qua iSCSI trông lớn hơn khi đọc.

### 4.2. Khi tăng QD

IOPS tăng theo QD vì mạng và target xử lý song song, và khoảng cách với local thu hẹp dần. Rand read: host đạt 59% local ở QD4, 76% ở QD8, 93% ở QD16, rồi giữ 87% ở QD32–QD128. VM thấp hơn host chút ít (khoảng 3–15% tùy QD; 97% của host ở mức đỉnh). Mixed 70/30 hội tụ về khoảng 51–52 nghìn IOPS đọc ở QD32 trở lên ở cả ba trục, nên khác biệt giữa các trục ở tải cao chủ yếu nằm ở latency đuôi, không phải IOPS.

### 4.3. Rand write: trần khoảng 30.000 IOPS

Host và VM đều giữ 29,5–31 nghìn IOPS (≈ 115–120 MiB/s) từ QD8 đến QD128, trong khi avg latency tăng tuyến tính theo QD (QD128: 4,16 ms và 4,26 ms, khớp với 128/30.000 IOPS). Đây là dấu hiệu của một điểm nghẽn cố định ở phía ổ hoặc target, không phải ở mạng hay virtio. Local cũng nằm quanh 30 nghìn ở QD8–QD64 nhưng có ba điểm vọt lên 55–57 nghìn (QD4, QD128) mà host và VM không tái hiện được, nên trạng thái cache ghi hoặc GC của SSD có thể đang ảnh hưởng. Chưa thể kết luận khi chưa lặp lại phép đo.

### 4.4. Latency đuôi (P99.99)

- Ở QD thấp, P99.99 tăng cùng chiều với latency trung bình qua từng lớp (rand read QD1: 0,214 → 0,823 → 1,81 ms).
- Ở QD128, rand write có P99.99 khoảng 27 ms (local), 32 ms (host) và 32 ms (VM): đuôi dài ở cả ba, trong đó phần lớn đã có ngay ở local.
- Ngược trực giác: ở QD128, mixed read có P99.99 là 13,4 ms (local), 9,9 ms (host) và 4,75 ms (VM), tức đường iSCSI cho đuôi ngắn hơn local. Chưa có đủ dữ liệu để giải thích (có thể liên quan hàng đợi của LIO hoặc cách I/O được gộp). Cần lặp lại để biết đây là hiện tượng ổn định hay nhiễu.
- P99.99 cần nhiều mẫu. Các lượt ghi ở QD thấp trên host và VM chỉ có vài nghìn IOPS, nên con số này chỉ mang tính tham khảo.

### 4.5. Tuần tự 1M

Host đạt 541 MiB/s (đọc) và 303 MiB/s (ghi). VM ghi đạt 356 MiB/s, cao hơn host (có thể do nhiễu hoặc khác biệt cache ghi). **VM đọc chỉ đạt 16 MiB/s với latency trung bình 489 ms và P99 hơn 2 giây.** Với cùng LUN, cùng đường mạng và cùng cấu hình `cache=none,io=native`, mức này quá thấp so với host và với kết quả rand read 4K của chính VM, nên nhiều khả năng là sự cố trong lượt đó (treo tạm thời, readahead, hoặc ảnh hưởng từ tải khác). Cần chạy lại trước khi dùng con số này.

## 5. Hạn chế và dữ liệu còn thiếu

1. **Network baseline chưa có:** chưa có ping và iperf3 (1 và 4 luồng) giữa com08 và com07. Chưa thể nói băng thông đọc cực đại \~297 MiB/s của host có phải do mạng hay không.
2. **Cấu hình cache của target:** giá trị `emulate_write_cache` của backstore `disk1` = 0 (write-through: LIO báo cho initiator là đĩa không có cache ghi). Số liệu ghi (`sync=0`) vẫn có thể phụ thuộc cache DRAM của chính SSD, vì LIO không thêm tầng cache ghi riêng ở chế độ này.
3. **Mỗi trục mới chạy một lần.** Các điểm bất thường (rand write local ở QD4 và QD128, seq read của VM, P99.99 mixed QD128) cần ít nhất một lần chạy lại.
4. **Trục A chưa đo seq 1M** (chạy không có `SEQ=1`).
5. **Không ghi nhận tiền xử lý (precondition).** Nếu một vùng đĩa chưa từng được ghi thì kết quả đọc có thể không đại diện cho trạng thái ổn định.
6. **Ba lần đo không cùng thời điểm và có thể không cùng host:** trục B chạy trên host initiator, trục C trên VM ở com08. LUN này com08 và com09 đều nhìn thấy, nên cần chắc chắn không có lượt nào chạy chồng lên nhau.
7. **Cấu hình VM trục C:** xem phụ lục B cho vCPU, RAM và các tham số đĩa (`cache=none`, `io=native`, virtio) để người khác lặp lại được.

## 6. Việc nên làm tiếp

1. Chạy iperf3 và ping (com08 → com07), điền vào bảng network baseline.
2. Ghi `emulate_write_cache` của `disk1` (`targetcli`, `get attribute emulate_write_cache`), giá trị hiện tại là 0. Đo thêm một lượt với `emulate_write_cache=1` và một lượt với `--sync=1` để so ghi bền vững.
3. Lặp lại trục A (tag `A_local_2`) và seq read của VM; so sánh để loại nhiễu.
4. Chạy trục A với `SEQ=1` để có số seq local.
5. Với P99.99, tăng `runtime` lên 300 s cho các QD ghi thấp nếu cần kết luận về đuôi latency.

## 7. Cần xác nhận

Tài liệu gốc có hai cách đặt tên trục B: mục 3 mô tả B là host initiator qua iSCSI (`com8 → com07 → disk`), còn bảng ở mục 5.2 ghi B là "VM local". Báo cáo này theo mục 3 (B = host qua iSCSI, đo tại `/dev/mapper/mpathb`), và coi bảng kết quả thứ hai là trục C (VM qua iSCSI). Nếu B thực chất là VM chạy local trên com07 thì phần phân tích 4.1 và 4.2 cần diễn giải lại. Hãy xác nhận trước khi dùng báo cáo.

## Phụ lục: script và cấu hình đã dùng

### A. Script `bench.sh`

Bản dưới đây là script chạy các lượt đo. Ở trục B và C, bản chạy thêm `// 0` trong bộ lọc `jq` để không lỗi khi thiếu giá trị percentile.

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

Cách chạy từng trục:

```bash
sudo ./bench.sh /dev/disk/by-id/wwn-0x... A_local 60          # trục A, trên com07
sudo SEQ=1 ./bench.sh /dev/mapper/mpathb B_iscsi_mpathb 60     # trục B, trên host initiator
sudo env SEQ=1 ./bench.sh /dev/vdb C_vm_vdb 60                 # trục C, trong VM
```

### B. Tạo VM cho trục C

```bash
export IMG=/var/lib/libvirt/images
export VM=vm1

sudo virt-install \
  --name $VM --ram 4096 --vcpus 4 \
  --disk path=$IMG/$VM.qcow2,format=qcow2,bus=virtio \
  --disk path=/dev/mapper/mpathb,format=raw,bus=virtio,cache=none,io=native,serial=benchdisk \
  --disk path=$IMG/$VM-seed.img,device=cdrom \
  --os-variant ubuntu24.04 \
  --network network=default,model=virtio \
  --graphics none --import --noautoconsole
```

| Thành phần | Giá trị |
| --- | --- |
| vCPU / RAM | 4 / 4096 MiB |
| OS | Ubuntu 24.04 (noble cloud image) |
| Đĩa OS | qcow2, virtio (`vda`) |
| Đĩa test | `/dev/mapper/mpathb`, raw, virtio (`vdb`), `cache=none`, `io=native` |
| Mạng | network `default`, virtio |



---

[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec    0             sender
[  5]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec                  receiver
[  7]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec    0             sender
[  7]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec                  receiver
[  9]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec    0             sender
[  9]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec                  receiver
[ 11]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec    0             sender
[ 11]   0.00-30.00  sec  8.20 GBytes  2.35 Gbits/sec                  receiver
[SUM]   0.00-30.00  sec  32.8 GBytes  9.39 Gbits/sec    0             sender
[SUM]   0.00-30.00  sec  32.8 GBytes  9.39 Gbits/sec                  receiver

[ ID] Interval           Transfer     Bitrate         Retr
[  5]   0.00-30.00  sec  32.7 GBytes  9.37 Gbits/sec    0             sender
[  5]   0.00-30.00  sec  32.7 GBytes  9.37 Gbits/sec                  receiver


10 packets transmitted, 10 received, 0% packet loss, time 9202ms
rtt min/avg/max/mdev = 0.189/0.213/0.236/0.015 ms
