# LAB: ĐO ĐẠC OVERHEAD MEMCPY TRONG LUỒNG I/O KVM/QEMU VIRTIO

![Pasted image 20261003223059](img/Pasted%20image%2020261003223059.png)

## 1. Mục tiêu thí nghiệm

- **đếm xem payload bị copy (memcpy) bao nhiêu lần** khi VM ghi 1 lệnh write đĩa qua KVM + virtio-blk

### 1.1 Hai kịch bản cache

**Kịch bản A: bật cache cả host lẫn VM**

```
RAM guest  ──copy──►  buffer của QEMU  ──copy──►  buffer kernel  ──DMA──► NIC/đĩa
```

- đây là bật cache cả host lẫn vm

**Kịch bản B: tắt cache, io = native (aio)**

```
RAM guest  ◄── mọi bên chỉ giữ con trỏ (iovec) trỏ vào đây ──  QEMU / kernel
                         │
                         └──DMA──► NIC/đĩa (đọc thẳng từ RAM guest)
```

- tắt cache , io = native ( aio )

### 1.2 Address space của process QEMU

```
Address space của process QEMU (trên host)
┌─────────────────────────────────────────────────┐
│ Vùng của QEMU (bình thường)                     │
│   - code, heap, stack                           │
│   - struct block layer, BlockDriverState        │
│   - struct VirtQueueElement, mảng iovec         │  ← chỉ chứa con trỏ + len
│   - thread pool, event loop...                  │
├─────────────────────────────────────────────────┤
│ Vùng RAM guest (mmap lớn, vd 8GB)               │
│   - kernel + app của guest                      │
│   - vring (desc / avail / used)                 │
│   - buffer request: header, data, status        │  ← dữ liệu thật ở đây
└─────────────────────────────────────────────────┘
```

![Pasted image 20261003234602](img/Pasted%20image%2020261003234602.png)

## 2. Thành phần

- KVM + QEMU
- `virtio-blk-pci` (hoặc `virtio-scsi-pci`
- VM : cache = none , io = native

## 3. Dựng môi trường

### 3.1 Tạo đĩa test và VM

```
sudo qemu-img create -f raw /var/lib/libvirt/images/test-io.raw 10G



sudo virt-install \
  --name lab-io-vm \
  --memory 4096 \
  --vcpus 2 \
  --cpu host-passthrough \
  --os-variant generic \
  --import \
  --disk path=/var/lib/libvirt/images/test-vm-root.qcow2,format=qcow2,bus=virtio \
  --disk path=/var/lib/libvirt/images/test-vm-seed.iso,device=cdrom \
  --disk path=/var/lib/libvirt/images/test-io.raw,format=raw,bus=scsi,cache=none,io=native \
  --controller type=scsi,model=virtio-scsi \
  --network network=default,model=virtio \
  --graphics none \
  --noautoconsole

  QEMU PID: 159321
```

### 3.2 Kiểm tra host đã tắt page cache với tiến trình qemu hiện tại

```
root@vhhl1c2lab2com07:/var/lib/libvirt/images# sudo ls -l /proc/$QEMU_PID/fd | grep "test-io.raw"
lrwx------ 1 root root 64 Oct  3 11:01 25 -> /var/lib/libvirt/images/test-io.raw
root@vhhl1c2lab2com07:/var/lib/libvirt/images# sudo cat /proc/$QEMU_PID/fdinfo/25
pos:    0
flags:  02140002
mnt_id: 807
lock:   1: OFDLCK ADVISORY  READ -1 08:05:18612805 100 101
lock:   2: OFDLCK ADVISORY  READ -1 08:05:18612805 201 201
```

## 4. Tìm PID của VM và dải RAM Guest trên Host

Lấy lại PID QEMU (lần đầu biến `$QEMU_PID` bị rỗng nên phải gán lại):

```
root@vhhl1c2lab2com07:~# echo "PID QEMU: $QEMU_PID"
PID QEMU:
root@vhhl1c2lab2com07:~# QEMU_PID=$(pgrep -f "lab-io.*test-io")
root@vhhl1c2lab2com07:~# QEMU_PID=$(pgrep -f "lab-io.*test-io")
root@vhhl1c2lab2com07:~# echo "PID QEMU: $QEMU_PID"
PID QEMU: 159321
```

Tìm dải địa chỉ RAM Guest (cách 1: grep trực tiếp, không ra kết quả):

```
root@vhhl1c2lab2com07:~# grep -E "rw-p.*00000000 00:00 0" /proc/$QEMU_PID/maps | awk '$2=="rw-p" {split($1,a,"-"); print "0x"a[1], "0x"a[2], ($5/1024/1024)"MB", $6}' | grep "4096MB"
root@vhhl1c2lab2com07:~# ^C
```

Cách 2: tính kích thước từng vùng trong `/proc/$QEMU_PID/maps` và `pmap`:

```
root@vhhl1c2lab2com07:~# while read range perms off dev inode path; do
>   s=$((0x${range%-*})); e=$((0x${range#*-}))
>   echo "$(( (e-s)/1024/1024 )) MB  $range  $perms  $path"
> done < /proc/$QEMU_PID/maps | sort -nr | head -5
4096 MB  7ff71be00000-7ff81be00000  rw-p
63 MB  7ff834021000-7ff838000000  ---p
63 MB  7ff82c021000-7ff830000000  ---p
63 MB  7ff824021000-7ff828000000  ---p
63 MB  7ff714021000-7ff718000000  ---p
root@vhhl1c2lab2com07:~# pmap -x $QEMU_PID | sort -k2 -nr | head -3
00007ff71be00000 4194304 1314816 1314816 rw---   [ anon ]
00007ff834021000   65404       0       0 -----   [ anon ]
00007ff82c021000   65404       0       0 -----   [ anon ]
root@vhhl1c2lab2com07:~#
```

(Chạy lại lần 2, kết quả giống hệt:)

```
root@vhhl1c2lab2com07:~# while read range perms off dev inode path; do
>   s=$((0x${range%-*})); e=$((0x${range#*-}))
>   echo "$(( (e-s)/1024/1024 )) MB  $range  $perms  $path"
> done < /proc/$QEMU_PID/maps | sort -nr | head -5
4096 MB  7ff71be00000-7ff81be00000  rw-p
63 MB  7ff834021000-7ff838000000  ---p
63 MB  7ff82c021000-7ff830000000  ---p
63 MB  7ff824021000-7ff828000000  ---p
63 MB  7ff714021000-7ff718000000  ---p
root@vhhl1c2lab2com07:~# pmap -x $QEMU_PID | sort -k2 -nr | head -3
00007ff71be00000 4194304 1314816 1314816 rw---   [ anon ]
00007ff834021000   65404       0       0 -----   [ anon ]
00007ff82c021000   65404       0       0 -----   [ anon ]
root@vhhl1c2lab2com07:~# ^C
```

→ Dải RAM Guest (4096 MB) nằm ở `7ff71be00000-7ff81be00000`.

## 5. Phương pháp thử 1: bẫy hàm `memcpy` bằng uprobe (không hiệu quả vì memcpy bị inline)

```
void *memcpy(void *dest, const void *src, size_t n);
```

**Nguyên lý đếm:** Ta đặt điều kiện lọc:

- Nếu `arg2 >= 4096`: Ghi nhận là **Payload Copy**
    
- Nếu `arg2 < 4096`: Ghi nhận là **Metadata Overhead** (bình thường).
    

```
uprobe:/lib/x86_64-linux-gnu/libc.so.6:memcpy /pid == 159321/ {
    if (arg2 >= 4096) {
        printf("[CANH BAO - PAYLOAD COPY] Comm: %s | Size: %d bytes\n", comm, arg2);
        @payload_copies = count();
    } else {
        printf("[METADATA OVERHEAD] Comm: %s | Size: %d bytes\n", comm, arg2);
        @metadata_copies = count();
    }
    @size_distribution = hist(arg2);
}
'
Attaching 1 probe...



root@vhhl1c2lab2com07:~# ^C
```

Thử thêm bằng `perf report` (không thấy symbol nào khớp):

```
root@vhhl1c2lab2com07:~# sudo perf report -i perf.data --no-children --sort sym | grep -Ei 'memcpy|memmove|iovec|blockalign'
root@vhhl1c2lab2com07:~# ^C
```

## 6. Phương pháp chuẩn: Memory Pointer Tracking + kiểm tra tầng nhân Host

Để kiểm chứng thực nghiệm việc không hề có thao tác copy payload (`Zero-Copy`) khi thực hiện lệnh ghi, phương pháp chuẩn xác nhất không phải là cố bẫy hàm `memcpy` bị inline, mà là **chứng minh bằng quan hệ địa chỉ con trỏ bộ nhớ (Memory Pointer Tracking)** kết hợp **kiểm tra hàm copy ở tầng nhân Host**.

Quy trình thực nghiệm gồm 2 bài test dưới đây:

### Bài test 1: Soi con trỏ `iov_base` trong `io_submit` với bảng phân trang RAM (Chứng minh QEMU không dùng Bounce Buffer)

Nếu QEMU không copy dữ liệu, địa chỉ bộ đệm mà QEMU nạp vào lệnh `io_submit` (`iov_base`) bắt buộc phải **nằm lọt thỏm bên trong dải RAM vật lý của máy ảo** được ánh xạ trên Host.

#### Bước 1: Tìm PID của VM và dải RAM Guest trên Host

Chạy trên Host:

```
# 1. Lấy PID của QEMU
QEMU_PID=$(pgrep -f "lab-io.*test-io")
echo "PID QEMU: $QEMU_PID"

# 2. Tìm dải địa chỉ anonymous mmap tương ứng với 4GB RAM của Guest
grep -E "rw-p.*00000000 00:00 0" /proc/$QEMU_PID/maps | awk '$2=="rw-p" {split($1,a,"-"); print "0x"a[1], "0x"a[2], ($5/1024/1024)"MB", $6}' | grep "4096MB"
```

_Kết quả sẽ trả về một dải địa chỉ Hex, ví dụ:_

`0x7f9a40000000 0x7f9b40000000 4096MB` $\rightarrow$ Đây chính là toàn bộ RAM 4GB của máy ảo trên không gian ảo của Host (HVA).

#### Bước 2: Bắt con trỏ `iov_base` khi QEMU submit I/O

Mở terminal trên Host và dùng `strace` để tóm đúng system call `io_submit`:

Bash

```
sudo strace -p $QEMU_PID -f -e trace=io_submit -s 128
```

_(Terminal sẽ đứng chờ request)_.

#### Bước 3: Phát lệnh ghi 4KB trong Guest VM

Mở console vào Guest VM (`virtio-blk` hoặc `virtio-scsi`):

Bash

```
sudo dd if=/dev/zero of=/dev/vdb bs=4096 count=1 oflag=direct
```

#### Bước 4: Đối chiếu kết quả

Quay lại terminal Host, `strace` sẽ in ra dòng dạng:

Plaintext

```
[pid 159321] io_submit(0x7f9b..., 1, [{..., aio_buf=[{iov_base=0x7f9a4210a000, iov_len=4096}], ...}]) = 1
```

- **Phân tích:**
    
    - Địa chỉ nạp vào kernel: `iov_base = 0x7f9a4210a000`.
        
    - So sánh với dải RAM Guest ở Bước 1: `0x7f9a40000000 <= 0x7f9a4210a000 <= 0x7f9b40000000`.
        
- **Kết luận thực nghiệm:** Con trỏ dữ liệu được gửi xuống Kernel Host **chính là địa chỉ nằm trong RAM của Guest**. QEMU hoàn toàn không cấp phát bất kỳ bộ đệm trung gian (Bounce Buffer) nào trong Userspace để copy 4KB này.
    

### Kết quả thực tế của Bài test 1 (strace trên lab)

```
root@vhhl1c2lab2com07:~# sudo strace -p $QEMU_PID -f -e trace=io_submit -s 128
strace: Process 159321 attached with 5 threads
strace: Process 164090 attached
strace: Process 164092 attached
strace: Process 164093 attached
strace: Process 164094 attached
strace: Process 164095 attached
strace: Process 164096 attached
strace: Process 164097 attached
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PWRITEV, aio_fildes=25, aio_buf=[{iov_base="\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., iov_len=4096}], aio_offset=0, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7df9e6000, iov_len=4096}], aio_offset=0, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7ddff6000, iov_len=4096}], aio_offset=4096, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dd7d4000, iov_len=4096}], aio_offset=12288, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff3d000, iov_len=4096}], aio_offset=10737352704, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de1e5000, iov_len=4096}], aio_offset=10737410048, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dfef8000, iov_len=4096}], aio_offset=0, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de10d000, iov_len=4096}], aio_offset=4096, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e241a000, iov_len=4096}], aio_offset=10737414144, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e45df000, iov_len=4096}], aio_offset=10737283072, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e4725000, iov_len=4096}], aio_offset=10737385472, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e523b000, iov_len=4096}], aio_offset=10737287168, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dffa4000, iov_len=4096}], aio_offset=10737213440, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de20f000, iov_len=4096}], aio_offset=10737115136, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e20bb000, iov_len=4096}], aio_offset=10737070080, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de4bb000, iov_len=4096}], aio_offset=10737041408, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dd990000, iov_len=4096}], aio_offset=10736951296, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e5446000, iov_len=4096}], aio_offset=10736918528, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e2051000, iov_len=4096}], aio_offset=10736910336, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dffa5000, iov_len=4096}], aio_offset=10736930816, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dfe35000, iov_len=4096}], aio_offset=10735837184, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e5c0b000, iov_len=4096}], aio_offset=16384, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e02b4000, iov_len=4096}], aio_offset=32768, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff44000, iov_len=4096}], aio_offset=65536, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff40000, iov_len=4096}], aio_offset=131072, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff41000, iov_len=4096}], aio_offset=262144, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff42000, iov_len=4096}], aio_offset=524288, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e25c8000, iov_len=4096}], aio_offset=1048576, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dd9ba000, iov_len=4096}], aio_offset=2097152, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de302000, iov_len=4096}], aio_offset=4194304, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de303000, iov_len=4096}], aio_offset=12288, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de304000, iov_len=4096}], aio_offset=28672, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de305000, iov_len=4096}], aio_offset=61440, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 4, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff3e000, iov_len=4096}], aio_offset=8192, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de307000, iov_len=4096}, {iov_base=0x7ff7e1958000, iov_len=4096}], aio_offset=20480, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e1959000, iov_len=20480}, {iov_base=0x7ff7e6662000, iov_len=4096}], aio_offset=36864, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e195e000, iov_len=4096}, {iov_base=0x7ff7e0aad000, iov_len=12288}, {iov_base=0x7ff7e1587000, iov_len=4096}, {iov_base=0x7ff7dd97a000, iov_len=4096}, {iov_base=0x7ff7e0ab8000, iov_len=32768}, {iov_base=0x7ff7de300000, iov_len=4096}], aio_offset=69632, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 4
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de301000, iov_len=4096}, {iov_base=0x7ff7dff33000, iov_len=4096}, {iov_base=0x7ff7df3b8000, iov_len=4096}, {iov_base=0x7ff7dff10000, iov_len=32768}, {iov_base=0x7ff7e0aa8000, iov_len=20480}, {iov_base=0x7ff7e2998000, iov_len=16384}, {iov_base=0x7ff7de0f8000, iov_len=16384}, {iov_base=0x7ff7de7f4000, iov_len=16384}, {iov_base=0x7ff7dff30000, iov_len=12288}], aio_offset=135168, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 1, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e015e000, iov_len=8192}, {iov_base=0x7ff7e15b0000, iov_len=16384}, {iov_base=0x7ff7dfef0000, iov_len=4096}, {iov_base=0x7ff7e54fb000, iov_len=4096}, {iov_base=0x7ff7dfef1000, iov_len=12288}, {iov_base=0x7ff7e173c000, iov_len=16384}, {iov_base=0x7ff7e1159000, iov_len=12288}, {iov_base=0x7ff7de0c4000, iov_len=4096}, {iov_base=0x7ff7e1f6f000, iov_len=4096}, {iov_base=0x7ff7de0c5000, iov_len=12288}, {iov_base=0x7ff7e0170000, iov_len=12288}, {iov_base=0x7ff7e25cb000, iov_len=4096}, {iov_base=0x7ff7e0173000, iov_len=4096}, {iov_base=0x7ff7e015c000, iov_len=8192}, {iov_base=0x7ff7ddf26000, iov_len=8192}, {iov_base=0x7ff7de2ac000, iov_len=16384}, {iov_base=0x7ff7e013c000, iov_len=16384}, {iov_base=0x7ff7de538000, iov_len=16384}, {iov_base=0x7ff7e1158000, iov_len=4096}, {iov_base=0x7ff7e02b7000, iov_len=4096}, {iov_base=0x7ff7e02e0000, iov_len=4096}, {iov_base=0x7ff7de541000, iov_len=4096}, {iov_base=0x7ff7e02e1000, iov_len=4096}, {iov_base=0x7ff7dd807000, iov_len=4096}, {iov_base=0x7ff7e02e2000, iov_len=8192}, {iov_base=0x7ff7e45eb000, iov_len=4096}, {iov_base=0x7ff7e0164000, iov_len=16384}, {iov_base=0x7ff7e01ca000, iov_len=4096}, {iov_base=0x7ff7ddf24000, iov_len=8192}, {iov_base=0x7ff7dfeb8000, iov_len=12288}], aio_offset=266240, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 1
[pid 159321] io_submit(0x7ff83a932000, 4, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de004000, iov_len=16384}], aio_offset=10736893952, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e243e000, iov_len=4096}], aio_offset=10736914432, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e0ac0000, iov_len=4096}, {iov_base=0x7ff7dd8db000, iov_len=4096}], aio_offset=10736922624, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de708000, iov_len=4096}, {iov_base=0x7ff7e0ac4000, iov_len=4096}, {iov_base=0x7ff7e02b5000, iov_len=8192}], aio_offset=10736934912, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 4
[pid 159321] io_submit(0x7ff83a932000, 4, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de31d000, iov_len=12288}, {iov_base=0x7ff7dffb4000, iov_len=16384}, {iov_base=0x7ff7e0144000, iov_len=16384}, {iov_base=0x7ff7dff0c000, iov_len=16384}, {iov_base=0x7ff7dfee6000, iov_len=8192}, {iov_base=0x7ff7de318000, iov_len=8192}, {iov_base=0x7ff7dd8ec000, iov_len=8192}], aio_offset=10736955392, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff34000, iov_len=8192}, {iov_base=0x7ff7de7f8000, iov_len=8192}, {iov_base=0x7ff7e0122000, iov_len=8192}], aio_offset=10737045504, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7de7ee000, iov_len=8192}, {iov_base=0x7ff7de31c000, iov_len=4096}, {iov_base=0x7ff7e2438000, iov_len=4096}, {iov_base=0x7ff7dff5b000, iov_len=4096}, {iov_base=0x7ff7dff23000, iov_len=4096}, {iov_base=0x7ff7e243a000, iov_len=8192}, {iov_base=0x7ff7e0ace000, iov_len=8192}], aio_offset=10737074176, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dff58000, iov_len=8192}, {iov_base=0x7ff7dc7c6000, iov_len=8192}, {iov_base=0x7ff7e5236000, iov_len=8192}, {iov_base=0x7ff7e499a000, iov_len=8192}, {iov_base=0x7ff7e2782000, iov_len=4096}], aio_offset=10737119232, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 4
[pid 159321] io_submit(0x7ff83a932000, 2, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e5bce000, iov_len=4096}, {iov_base=0x7ff7e47e0000, iov_len=4096}, {iov_base=0x7ff7dc7c5000, iov_len=4096}, {iov_base=0x7ff7de2b0000, iov_len=4096}, {iov_base=0x7ff7e270e000, iov_len=4096}, {iov_base=0x7ff7dd98e000, iov_len=4096}, {iov_base=0x7ff7e478a000, iov_len=4096}, {iov_base=0x7ff7e282d000, iov_len=4096}, {iov_base=0x7ff7e0160000, iov_len=4096}, {iov_base=0x7ff7e47c5000, iov_len=4096}, {iov_base=0x7ff7de3e8000, iov_len=4096}, {iov_base=0x7ff7de6f5000, iov_len=4096}, {iov_base=0x7ff7de7f3000, iov_len=4096}, {iov_base=0x7ff7dff36000, iov_len=4096}], aio_offset=10737156096, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e5a39000, iov_len=4096}, {iov_base=0x7ff7dff1e000, iov_len=4096}, {iov_base=0x7ff7dd8bf000, iov_len=4096}, {iov_base=0x7ff7e5ab6000, iov_len=4096}, {iov_base=0x7ff7de6d5000, iov_len=4096}, {iov_base=0x7ff7dff54000, iov_len=4096}, {iov_base=0x7ff7e48d6000, iov_len=4096}, {iov_base=0x7ff7e5ee7000, iov_len=4096}, {iov_base=0x7ff7de70e000, iov_len=4096}, {iov_base=0x7ff7e45db000, iov_len=4096}, {iov_base=0x7ff7dc7c0000, iov_len=4096}, {iov_base=0x7ff7e1165000, iov_len=4096}, {iov_base=0x7ff7de236000, iov_len=4096}, {iov_base=0x7ff7de2c4000, iov_len=4096}, {iov_base=0x7ff7e15a1000, iov_len=4096}, {iov_base=0x7ff7e47c7000, iov_len=4096}], aio_offset=10737217536, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 2
[pid 159321] io_submit(0x7ff83a932000, 3, [{aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e159f000, iov_len=4096}, {iov_base=0x7ff7e299e000, iov_len=4096}, {iov_base=0x7ff7de6de000, iov_len=4096}, {iov_base=0x7ff7dd8cb000, iov_len=4096}, {iov_base=0x7ff7e0177000, iov_len=4096}, {iov_base=0x7ff7dc019000, iov_len=4096}, {iov_base=0x7ff7e1df1000, iov_len=4096}, {iov_base=0x7ff7e204b000, iov_len=4096}, {iov_base=0x7ff7e4638000, iov_len=4096}, {iov_base=0x7ff7de0bd000, iov_len=4096}, {iov_base=0x7ff7dd9e9000, iov_len=4096}, {iov_base=0x7ff7e0b1b000, iov_len=4096}, {iov_base=0x7ff7e5efb000, iov_len=4096}, {iov_base=0x7ff7e0acb000, iov_len=4096}, {iov_base=0x7ff7de6e1000, iov_len=4096}], aio_offset=10737291264, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7dc715000, iov_len=4096}, {iov_base=0x7ff7e5231000, iov_len=4096}, {iov_base=0x7ff7e666f000, iov_len=4096}, {iov_base=0x7ff7dc0f4000, iov_len=4096}, {iov_base=0x7ff7e5f55000, iov_len=4096}, {iov_base=0x7ff7ddfed000, iov_len=4096}, {iov_base=0x7ff7e4654000, iov_len=4096}], aio_offset=10737356800, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}, {aio_data=0, aio_lio_opcode=IOCB_CMD_PREADV, aio_fildes=25, aio_buf=[{iov_base=0x7ff7e44da000, iov_len=4096}, {iov_base=0x7ff7e4458000, iov_len=4096}, {iov_base=0x7ff7e116a000, iov_len=4096}, {iov_base=0x7ff7e15e2000, iov_len=4096}, {iov_base=0x7ff7e0ab1000, iov_len=4096}], aio_offset=10737389568, aio_flags=IOCB_FLAG_RESFD, aio_resfd=14}]) = 3
[pid 164095] +++ exited with 0 +++
[pid 164090] +++ exited with 0 +++
[pid 164097] +++ exited with 0 +++
[pid 164094] +++ exited with 0 +++
[pid 164096] +++ exited with 0 +++
[pid 164092] +++ exited with 0 +++
[pid 164093] +++ exited with 0 +++
strace: Process 164099 attached
strace: Process 164100 attached
strace: Process 164101 attached
strace: Process 164102 attached
strace: Process 164103 attached
strace: Process 164104 attached
strace: Process 164105 attached
strace: Process 164106 attached




[pid 164102] +++ exited with 0 +++
[pid 164100] +++ exited with 0 +++
[pid 164099] +++ exited with 0 +++
[pid 164105] +++ exited with 0 +++
[pid 164106] +++ exited with 0 +++
[pid 164103] +++ exited with 0 +++
[pid 164101] +++ exited with 0 +++
[pid 164104] +++ exited with 0 +++
```

### Bài test 2: Kiểm tra hàm ghim trang / submit bio ở tầng nhân Host bằng bpftrace

Lỗi trên xảy ra do khi paste nhiều dòng vào terminal, dấu nháy đơn `'` bị ngắt quãng khiến bash hiểu nhầm các dòng lệnh của `bpftrace` thành các lệnh shell độc lập (gây ra lỗi `command not found` và `syntax error`).

#### Cách 1: Ghi ra file `.bt` rồi chạy

```
cat << 'EOF' > test_zerocopy.bt
kprobe:bio_iov_iter_get_pages {
    if (pid == $1) {
        printf("[ZERO-COPY] bio_iov_iter_get_pages called (PID: %d)\n", pid);
    }
}

tracepoint:block:block_rq_issue {
    if (args->bytes >= 4096) {
        printf("[DISK I/O] Request: %d bytes (comm: %s)\n", args->bytes, comm);
    }
}
EOF

sudo bpftrace test_zerocopy.bt $QEMU_PID
```

#### Cách 2: Chạy lệnh trên đúng 1 dòng duy nhất (One-liner)

Nếu muốn chạy trực tiếp bằng tham số `-e`, dán nguyên dòng sau:

Bash

```
sudo bpftrace -e 'kprobe:bio_iov_iter_get_pages /pid == '$QEMU_PID'/ { printf("[ZERO-COPY] bio_iov_iter_get_pages (PID: %d)\n", pid); } tracepoint:block:block_rq_issue { if (args->bytes >= 4096) { printf("[DISK I/O] Request: %d bytes (comm: %s)\n", args->bytes, comm); } }'
```

#### Xác nhận hoạt động

1. Khi lệnh chạy thành công, terminal sẽ hiện:
    
    Plaintext
    
2. Sang Guest VM gõ lệnh ghi đĩa:
    
    Bash
    
    ```
    sudo dd if=/dev/zero of=/dev/vdb bs=4096 count=1 oflag=direct
    ```
    
3. Xem log trả về ngay lập tức trên Host để đối chiếu.
    

#### Script `cw.bt` (đoạn đã paste, phần đầu bị cắt) và kết quả chạy thực tế

```
   kprobe:get_user_pages_fast /@in_aio_write[tid]/ {
> }
>
> tracepoint:syscalls:sys_exit_io_submit /@in_io_submit[tid]/ {
>     delete(@in_io_submit[tid]);
> }
>
> kprobe:aio_write /@in_io_submit[tid]/ {
>     @in_aio_write[tid] = 1;
> }
>
> kretprobe:aio_write /@in_aio_write[tid]/ {
>     delete(@in_aio_write[tid]);
> }
>
> // Bắt hàm ghim trang RAM vật lý
> kprobe:get_user_pages_fast /@in_aio_write[tid]/ {
>     printf("[PIN-PAGE] get_user_pages_fast: HVA=0x%lx | Pages=%d | TID=%d\n", arg0, arg1, tid);
>     @pin_pages = count();
> }
>
> // Bắt thời điểm bio được đẩy xuống Driver ổ cứng
> kprobe:submit_bio /@in_aio_write[tid]/ {
>     printf("[SUBMIT-BIO] Day bio xuong Hardware DMA (TID=%d)\n", tid);
>     @bios = count();
> }
> EOF
root@vhhl1c2lab2com07:~# sudo bpftrace cw.bt $QEMU_PID
Attaching 6 probes...
[PIN-PAGE] get_user_pages_fast: HVA=0x7ff80b752000 | Pages=1 | TID=159321
[SUBMIT-BIO] Day bio xuong Hardware DMA (TID=159321)

^C

@bios: 1



@pin_pages: 1
```

## 7. Tài liệu tham khảo

[Understanding Disk I/O in QEMU/KVM with VirtIO | Veeam Community Resource Hub](https://community.veeam.com/blogs-and-podcasts-57/understanding-disk-i-o-in-qemu-kvm-with-virtio-13460)



----
Trace chi tiết từng lệnh : 
```
sudo strace dd if=/dev/zero of=/dev/sda bs=4096 count=1 oflag=direct

```

```
ubuntu@ubuntu:~$ sudo strace dd if=/dev/zero of=/dev/sda bs=4096 count=1 oflag=direct
execve("/usr/bin/dd", ["dd", "if=/dev/zero", "of=/dev/sda", "bs=4096", "count=1", "oflag=direct"], 0x7ffce0447fc8 /* 19 vars */) = 0
brk(NULL)                               = 0x564868b68000
arch_prctl(0x3001 /* ARCH_??? */, 0x7ffe31a95580) = -1 EINVAL (Invalid argument)
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6724a76000
access("/etc/ld.so.preload", R_OK)      = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=21228, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 21228, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a70000
close(3)                                = 0
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\237\2\0\0\0\0\0"..., 832) = 832
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
pread64(3, "\4\0\0\0 \0\0\0\5\0\0\0GNU\0\2\0\0\300\4\0\0\0\3\0\0\0\0\0\0\0"..., 48, 848) = 48
pread64(3, "\4\0\0\0\24\0\0\0\3\0\0\0GNU\0jl\265\250\17\325\345\273[\363\256\252\21\32\335\230"..., 68, 896) = 68
newfstatat(3, "", {st_mode=S_IFREG|0755, st_size=2220400, ...}, AT_EMPTY_PATH) = 0
pread64(3, "\6\0\0\0\4\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0@\0\0\0\0\0\0\0"..., 784, 64) = 784
mmap(NULL, 2264656, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f6724847000
mprotect(0x7f672486f000, 2023424, PROT_NONE) = 0
mmap(0x7f672486f000, 1658880, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f672486f000
mmap(0x7f6724a04000, 360448, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bd000) = 0x7f6724a04000
mmap(0x7f6724a5d000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x215000) = 0x7f6724a5d000
mmap(0x7f6724a63000, 52816, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f6724a63000
close(3)                                = 0
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6724844000
arch_prctl(ARCH_SET_FS, 0x7f6724844740) = 0
set_tid_address(0x7f6724844a10)         = 2884
set_robust_list(0x7f6724844a20, 24)     = 0
rseq(0x7f67248450e0, 0x20, 0, 0x53053053) = 0
mprotect(0x7f6724a5d000, 16384, PROT_READ) = 0
mprotect(0x564862500000, 4096, PROT_READ) = 0
mprotect(0x7f6724ab0000, 8192, PROT_READ) = 0
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f6724a70000, 21228)           = 0
rt_sigaction(SIGINT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGUSR1, {sa_handler=0x5648624f48a0, sa_mask=[INT USR1], sa_flags=SA_RESTORER, sa_restorer=0x7f6724889520}, NULL, 8) = 0
rt_sigaction(SIGINT, {sa_handler=0x5648624f4890, sa_mask=[INT USR1], sa_flags=SA_RESTORER|SA_NODEFER|SA_RESETHAND|0xffffffff00000000, sa_restorer=0x7f6724889520}, NULL, 8) = 0
getrandom("\x65\x02\xb6\x3e\x1a\x1e\x62\x1e", 8, GRND_NONBLOCK) = 8
brk(NULL)                               = 0x564868b68000
brk(0x564868b89000)                     = 0x564868b89000
openat(AT_FDCWD, "/usr/lib/locale/locale-archive", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=3048928, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 3048928, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f672455b000
close(3)                                = 0
openat(AT_FDCWD, "/usr/share/locale/locale.alias", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=2996, ...}, AT_EMPTY_PATH) = 0
read(3, "# Locale name alias data base.\n#"..., 4096) = 2996
read(3, "", 4096)                       = 0
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_IDENTIFICATION", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_IDENTIFICATION", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=258, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 258, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724aaf000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/x86_64-linux-gnu/gconv/gconv-modules.cache", O_RDONLY) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=27002, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 27002, PROT_READ, MAP_SHARED, 3, 0) = 0x7f6724554000
close(3)                                = 0
futex(0x7f6724a62a6c, FUTEX_WAKE_PRIVATE, 2147483647) = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_MEASUREMENT", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_MEASUREMENT", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=23, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 23, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a75000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_TELEPHONE", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_TELEPHONE", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=47, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 47, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a74000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_ADDRESS", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_ADDRESS", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=127, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 127, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a73000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_NAME", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_NAME", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=62, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 62, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a72000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_PAPER", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_PAPER", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=34, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 34, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a71000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_MESSAGES", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_MESSAGES", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFDIR|0755, st_size=4096, ...}, AT_EMPTY_PATH) = 0
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_MESSAGES/SYS_LC_MESSAGES", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=48, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 48, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a70000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_MONETARY", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_MONETARY", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=270, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 270, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724553000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_COLLATE", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_COLLATE", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=1406, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 1406, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724552000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_TIME", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_TIME", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=3360, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 3360, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724551000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_NUMERIC", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_NUMERIC", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=50, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 50, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724550000
close(3)                                = 0
openat(AT_FDCWD, "/usr/lib/locale/C.UTF-8/LC_CTYPE", O_RDONLY|O_CLOEXEC) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/lib/locale/C.utf8/LC_CTYPE", O_RDONLY|O_CLOEXEC) = 3
newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=353616, ...}, AT_EMPTY_PATH) = 0
mmap(NULL, 353616, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f67244f9000
close(3)                                = 0
openat(AT_FDCWD, "/dev/zero", O_RDONLY) = 3
dup2(3, 0)                              = 0
close(3)                                = 0
lseek(0, 0, SEEK_CUR)                   = 0
openat(AT_FDCWD, "/dev/sda", O_WRONLY|O_CREAT|O_TRUNC|O_DIRECT, 0666) = 3
dup2(3, 1)                              = 1
close(3)                                = 0
read(0, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
write(1, "\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0\0"..., 4096) = 4096
close(0)                                = 0
close(1)                                = 0
openat(AT_FDCWD, "/usr/share/locale/C.UTF-8/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/C.utf8/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale/C/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale-langpack/C.UTF-8/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale-langpack/C.utf8/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
openat(AT_FDCWD, "/usr/share/locale-langpack/C/LC_MESSAGES/coreutils.mo", O_RDONLY) = -1 ENOENT (No such file or directory)
write(2, "1+0 records in\n1+0 records out\n", 311+0 records in
1+0 records out
) = 31
write(2, "4096 bytes (4.1 kB, 4.0 KiB) cop"..., 614096 bytes (4.1 kB, 4.0 KiB) copied, 0.000361159 s, 11.3 MB/s) = 61
write(2, "\n", 1
)                       = 1
close(2)                                = 0
exit_group(0)                           = ?
+++ exited with 0 +++

```
Nhóm 1: Khởi động chương trình & Xin cấp phát RAM
- hàm execve -> khởi tạo lệnh ( execve , exec , vector of argurments , enviroments )
- `brk` -  Kiểm tra ranh giới bộ nhớ

```
brk(NULL) = 0x564868b68000

  Hàm `brk`** (_break_): Hàm thay đổi kích thước của vùng nhớ **Heap** (vùng nhớ dùng để cấp phát động cho chương trình). , phần này ở HEAP ( read only )
```
- Dòng 4: Cấp phát vùng RAM trống đầu tiên
```
mmap(NULL, 8192, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6724a76000
```
Dòng 5: Kiểm tra file cấu hình ưu tiên

```
access("/etc/ld.so.preload", R_OK) = -1 ENOENT (No such file or directory)
```
Dòng 6 đến 9: Mở và đọc Sổ địa chỉ thư viện

```
openat(AT_FDCWD, "/etc/ld.so.cache", O_RDONLY|O_CLOEXEC) = 3 newfstatat(3, "", {st_mode=S_IFREG|0644, st_size=21228, ...}, AT_EMPTY_PATH) = 0 
mmap(NULL, 21228, PROT_READ, MAP_PRIVATE, 3, 0) = 0x7f6724a70000 close(3) = 0
```
- **Ý nghĩa từng dòng:**
    - **`openat`**: Mở file `/etc/ld.so.cache` (Sổ địa chỉ chứa vị trí các thư viện). Trả về mã định danh file (File Descriptor) là **`3`**.
    - **`newfstatat`**: Kiểm tra thông tin file số `3`. Kết quả cho biết đây là file thông thường (`S_IFREG`), quyền `0644`, kích thước `21228` bytes.
    - **`mmap`**: Ánh xạ toàn bộ `21228` bytes của file số `3` này vào RAM với quyền chỉ đọc (`PROT_READ`) để tra cứu cho nhanh.
    - **`close(3)`**: Đóng file số `3` lại vì đã ánh xạ xong vào RAM, giải phóng tài nguyên.

Dòng 10 đến 16: Mở và đọc cấu trúc Thư viện C (`libc.so.6`)
```
openat(AT_FDCWD, "/lib/x86_64-linux-gnu/libc.so.6", O_RDONLY|O_CLOEXEC) = 3
 read(3, "\177ELF\2\1\1\3\0\0\0\0\0\0\0\0\3\0>\0\1\0\0\0P\237\2\0\0\0\0\0"..., 832) = 832 
 pread64(3, "...", 784, 64) = 784 pread64(3, "...", 48, 848) = 48 pread64(3, "...", 68, 896) = 68 
 newfstatat(3, "", {st_mode=S_IFREG|0755, st_size=2220400, ...}, AT_EMPTY_PATH) = 0
  pread64(3, "...", 784, 64) = 784
```

Dòng 17 đến 22: Phân chia các phân vùng bảo mật cho Thư viện C trên RAM
```
mmap(NULL, 2264656, PROT_READ, MAP_PRIVATE|MAP_DENYWRITE, 3, 0) = 0x7f6724847000 mprotect(0x7f672486f000, 2023424, PROT_NONE) = 0 mmap(0x7f672486f000, 1658880, PROT_READ|PROT_EXEC, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x28000) = 0x7f672486f000 mmap(0x7f6724a04000, 360448, PROT_READ, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x1bd000) = 0x7f6724a04000 mmap(0x7f6724a5d000, 24576, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_DENYWRITE, 3, 0x215000) = 0x7f6724a5d000 mmap(0x7f6724a63000, 52816, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_FIXED|MAP_ANONYMOUS, -1, 0) = 0x7f6724a63000 close(3) = 0
```
- **Ý nghĩa:** Hệ thống tiến hành nạp thư viện C vào RAM tại địa chỉ `0x7f6724847000`.
- **`mprotect`** (_memory protection_) đặt vùng nhớ hơn 2MB này về trạng thái `PROT_NONE` (Không cho ai làm gì cả) để xóa trắng an toàn, sau đó dùng tiếp các lệnh `mmap` kèm cờ `MAP_FIXED` để chia nhỏ vùng nhớ này thành các khu vực bảo mật:
    - Khu vực 1 (`PROT_READ|PROT_EXEC`): Chỉ cho **Đọc và Thực thi** mã lệnh của thư viện C (Cấm ghi đè để tránh virus).
    - Khu vực 2 (`PROT_READ`): Chỉ cho **Đọc** dữ liệu hằng số.
    - Khu vực 3 (`PROT_READ|PROT_WRITE`): Cho phép **Đọc và Ghi** các biến dữ liệu động của thư viện.
    - Cuối cùng, `close(3)` đóng file thư viện C lại vì toàn bộ ruột của nó đã được phân bổ an toàn trên RAM.
Dòng 24 đến 28: Khởi tạo Quản lý Luồng (Thread) cho tiến trình `dd`
```
mmap(NULL, 12288, PROT_READ|PROT_WRITE, MAP_PRIVATE|MAP_ANONYMOUS, -1, 0) = 0x7f6724844000 arch_prctl(ARCH_SET_FS, 0x7f6724844740) = 0 set_tid_address(0x7f6724844a10) = 2884 set_robust_list(0x7f6724844a20, 24) = 0 rseq(0x7f67248450e0, 0x20, 0, 0x53053053) = 0
```
Dòng 29 đến 31: Khóa bảo mật bộ nhớ (Hậu nạp thư viện)
```
mprotect(0x7f6724a5d000, 16384, PROT_READ) = 0 mprotect(0x564862500000, 4096, PROT_READ) = 0 mprotect(0x7f6724ab0000, 8192, PROT_READ) = 0
```
**Ý nghĩa:** Chuyển các vùng nhớ chứa bảng tra cứu địa chỉ hàm (bảng GOT/PLT) sang chế độ Chỉ đọc (`PROT_READ`). Cơ chế này gọi là **RELRO (Relocation Read-Only)**, ngăn chặn hacker ghi đè địa chỉ hàm để hack hệ thống
Dòng 32 đến 36: Thiết lập Giới hạn tài nguyên và Tín hiệu ngắt
```
prlimit64(0, RLIMIT_STACK, NULL, {rlim_cur=8192*1024, rlim_max=RLIM64_INFINITY}) = 0
munmap(0x7f6724a70000, 21228) = 0
rt_sigaction(SIGINT, NULL, {sa_handler=SIG_DFL, sa_mask=[], sa_flags=0}, 8) = 0
rt_sigaction(SIGUSR1, {sa_handler=0x5648624f48a0, ...}, NULL, 8) = 0
rt_sigaction(SIGINT, {sa_handler=0x5648624f4890, ...}, NULL, 8) = 0

```
- **`prlimit64`**: Kiểm tra giới hạn bộ nhớ Stack của hệ thống (ở đây là 8MB - `8192*1024`).
- **`munmap`**: Hủy ánh xạ vùng nhớ 21228 bytes của file `ld.so.cache` ở đầu lệnh vì không cần dùng nữa, trả lại RAM cho máy.
- **`rt_sigaction`**: Cài đặt cách `dd` phản ứng khi nhận tín hiệu từ người dùng. Ví dụ: khi bạn nhấn `Ctrl + C` (`SIGINT`), hàm xử lý đặc biệt của `dd` sẽ bắt lấy để dừng ghi và in ra báo cáo ngay lập tức chứ không chết ngậm ngùi.
Dòng 37 đến 39: Lấy số ngẫu nhiên và mở rộng bộ nhớ Heap
```
Heapgetrandom("\x65\x02\xb6\x3e\x1a\x1e\x62\x1e", 8, GRND_NONBLOCK) = 8 brk(NULL) = 0x564868b68000 brk(0x564868b89000) = 0x564868b89000

```
Đoạn còn lại: Khởi tạo Ngôn ngữ và Định dạng (Locale) 
- Chuẩn bị đầu vào: Mở "vòi nước" `/dev/zero`
	```
  openat(AT_FDCWD, "/dev/zero", O_RDONLY) = 3
 dup2(3, 0) = 0
  close(3) = 0 
  lseek(0, 0, SEEK_CUR) = 0
	```
	- Mở đích đến ổ đĩa `/dev/sda`
	```
openat(AT_FDCWD, "/dev/sda", O_WRONLY|O_CREAT|O_TRUNC|O_DIRECT, 0666) = 3 dup2(3, 1) = 1 close(3) = 0
	```
	- **`openat`**: Mở ổ đĩa `/dev/sda` với một loạt cờ cấu hình quan trọng:
	- `O_WRONLY`: Chỉ ghi dữ liệu (_Write Only_).
	- `O_CREAT` và `O_TRUNC`: Tạo mới hoặc xóa trắng file nếu có sẵn (đối với ổ đĩa vật lý thì cờ này chỉ mang tính thủ tục).
	- **`O_DIRECT`**: **Cực kỳ quan trọng!**
		
		Đây chính là cờ kích hoạt chế độ **Direct I/O** (bỏ qua bộ nhớ đệm). Hệ điều hành sẽ ép dữ liệu phải đập thẳng xuống phần cứng ổ đĩa, cấm không cho lưu tạm trên RAM để tăng tính thực tế và an toàn cho việc ghi đĩa.
	- Kết quả: Trả về mã số file tạm thời là **`3`**.
		- **`dup2(3, 1)`**: `dd` ghi dữ liệu ra **Standard Output (Mã số là 1)**. Hàm này chuyển hướng Standard Output (số 1) thẳng vào file số 3 (`/dev/sda`).
	- **`close(3)`**: Đóng số 3 lại vì số 1 đã chịu trách nhiệm ghi vào `/dev/sda`
- Đọc và Ghi 4096 bytes số 0
	```
	read(0, "\0\0\0\0..."..., 4096) = 4096
	 write(1, "\0\0\0\0..."..., 4096) = 4096
	```
	- **`read(0, ..., 4096)`**: `dd` thò tay vào file số 0 (`/dev/zero`) để múc ra đúng `4096` bytes dữ liệu. Bạn có thể thấy chuỗi `\0\0\0\0...` chính là các ký tự null (số 0 trống rỗng). Kết quả trả về `= 4096` (đọc thành công 4KB).
	- **`write(1, ..., 4096)`**: Ngay lập tức, `dd` ném 4096 bytes số 0 vừa đọc được vào file số 1 (`/dev/sda`) để **ghi đè trực tiếp lên vị trí đầu tiên của ổ đĩa**. Kết quả trả về `= 4096` (đã ghi đè thành công, chính thức xóa sổ Master Boot Record hoặc bảng phân vùng của ổ đĩa).
	```
	openat(AT_FDCWD, "/usr/share/locale/.../coreutils.mo", O_RDONLY) = -1 ENOENT ...
	```
	Tìm kiếm file dịch ngôn ngữ (Thất bại)
In báo cáo kết quả ra màn hình Terminal

```
write(2, "1+0 records in\n1+0 records out\n", 31) = 31 write(2, "4096 bytes (4.1 kB, 4.0 KiB) cop"..., 61) = 61 write(2, "\n", 1) = 1 close(2)
```

