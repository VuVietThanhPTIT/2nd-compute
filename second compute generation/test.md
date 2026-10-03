

### PHẦN 1: 4 Tham số truyền vào Script

Cú pháp gọi script:

  

Bash

```
sudo ./bench-raw.sh <TARGET> <LABEL> [OFFSET] [SIZE]
#                    $1        $2       $3      $4
```

|**Tham số**|**Biến trong shell**|**Ý nghĩa**|**Ví dụ thực tế**|
|---|---|---|---|
|**1. TARGET**|`$1`|**Đường dẫn ổ đĩa thô hoặc file ảnh** mà bạn muốn `fio` đọc/ghi I/O vào.|`/dev/mapper/mpatha`, `/dev/vda`, hoặc `/var/target/disk1.img`|
|**2. LABEL**|`$2`|**Tên nhãn điểm đo** để phân biệt kết quả và đặt tên file JSON lưu tạm trong `/tmp/`.|`storage-node`, `kvm-mpath`, `vm-guest`|
|**3. OFFSET**|`$3` _(Tùy chọn)_|**Vị trí bắt đầu đọc/ghi** (bỏ qua bao nhiêu GB đầu của đĩa).|`25G` (nhảy qua 25GB đầu)|
|**4. SIZE**|`$4` _(Tùy chọn)_|**Độ dài không gian đĩa** dùng để chạy test (chỉ test trong phạm vi này).|`20G` (chỉ chạy trong 20GB tiếp theo)|

#### ❓ Tại sao lại cần `OFFSET` và `SIZE`? (Cực kỳ quan trọng khi test Raw)

Một ổ đĩa 50GB được tổ chức như một thước đo từ vạch `0 GB` đến `50 GB`:

  

Plaintext

```
[------------------ 50GB Block Device (/dev/vda) ------------------]
|-- OS & Data hệ thống (/dev/vda1) --|------- Vùng trống chưa dùng -------|
0GB                                 25GB                                45GB      50GB
                                     ^--- Bắt đầu ở đây ---- Độ dài 20GB ---^
                                          (OFFSET = 25G)       (SIZE = 20G)
```

1. **Nếu KHÔNG có Offset & Size (chạy mặc định):**
    - `fio` sẽ ghi đè ngẫu nhiên từ byte `0` trở đi 
    - Sector 0 chứa **Bảng phân vùng (Partition Table - GPT/MBR)** và các sector đầu chứa **Kernel Linux, Bootloader `/boot/efi`**.
    - Khi bạn chạy test WRITE thô, `fio` sẽ **xóa sạch hệ điều hành**, máy ảo sẽ báo lỗi I/O error và `Kernel Panic` ngay lập tức.
2. **Khi CÓ `OFFSET=25G` và `SIZE=20G`:** 
    - Bạn ra lệnh cho `fio`: _"Hãy nhảy qua 25GB đầu (nơi chứa OS), chỉ đọc/ghi thô vào dải từ GB thứ 25 đến GB thứ 45"_.
    - **Kết quả:** Vẫn test trực tiếp trên khối thô (Raw Block Device), bypass qua toàn bộ tầng filesystem/caching, nhưng **an toàn tuyệt đối cho hệ điều hành**.

### PHẦN 2: Các tham số cốt lõi bên trong lệnh `fio`

Bên trong hàm `run_fio()`, script đã dựng sẵn lệnh chạy với các tham số chuẩn benchmark quốc tế:

Bash

```
fio --name="${test_name}" \
    --filename="${TARGET}" \
    --direct=1 \
    --sync=0 \
    --ioengine=libaio \
    --rw="${rw}" \
    --bs="${bs}" \
    --iodepth="${qd}" \
    --numjobs="${jobs}" \
    --group_reporting \
    --time_based --runtime=30 --ramp_time=5 \
    --percentile_list=50:90:99:99.99
```

Bản chất từng tham số:

  

1. **`--direct=1` (Bắt buộc khi test Raw / Storage):**
      
    - Bỏ qua hoàn toàn bộ nhớ đệm (Page Cache / Buffer Cache) trên RAM của Linux.
        
          
        
    - Mọi lệnh đọc/ghi bắt buộc phải đẩy gói tin qua cáp mạng (iSCSI) hoặc xuống ổ đĩa thật. Không có cờ này, số đo IOPS sẽ lên đến hàng triệu vì bạn chỉ đang đọc/ghi trên RAM.
        
          
        
2. **`--ioengine=libaio`:**
    
      
    - Thư viện xử lý I/O bất đồng bộ (Linux Native Asynchronous I/O) của nhân Linux. Giúp gửi nhiều I/O cùng lúc mà tiến trình không bị block chờ kết quả.
        
          
        
3. **`--bs=4k` hoặc `--bs=1m` (Block Size):**
    
      
    - Kích thước của mỗi khối dữ liệu gửi đi.
        
          
        
    - **4K:** Kích thước tiêu chuẩn của database, OS boot (đo chỉ số **IOPS** và **Latency**).
        
          
        
    - **1M:** Kích thước khối dữ liệu lớn như sao chép file phim, backup (đo chỉ số **Bandwidth / Throughput**).
        
          
        
4. **`--iodepth=32` (Queue Depth):**
    
      
    - Độ sâu hàng đợi. Báo cho `fio` đẩy liên tục 32 request I/O cùng một lúc vào thiết bị lưu trữ mà không chờ cái trước xong mới gửi cái sau.
        
          
        
5. **`--numjobs=4`:**
    
      
    - Tạo 4 tiến trình (thread/process) chạy song song.
        
          
        
    - Tổng số I/O "đang bay" trên đường truyền lúc này = `iodepth x numjobs` = `32 x 4 = 128 request`.
        
          
        
6. **`--ramp_time=5` và `--runtime=30`:**
    
      
    - `--ramp_time=5`: Bỏ qua 5 giây đầu tiên khi chạy (giai đoạn khởi động, cache đang ấm, số liệu chưa chuẩn).
        
          
        
    - `--runtime=30`: Sau 5s làm nóng, `fio` ghi nhận kết quả chính xác trong 30 giây tiếp theo.
        
          
        
7. **`--percentile_list=50:90:99:99.99`:**
    
      
    - Ra lệnh cho `fio` xuất ra các mốc trễ quan trọng:
        
          
        - **P50:** 50% số request có trễ dưới mức này (trễ trung bình).
            
              
            
        - **P99:** 99% số request chạy nhanh hơn mức này.
            
              
            
        - **P99.99:** 0.01% request chậm nhất (dùng để phát hiện jitter mạng hoặc nghẽn iSCSI).
            
              
            

### PHẦN 3: Áp dụng vào 3 node của bạn

Tùy theo từng node, bạn truyền các tham số tương ứng như sau:

  

#### 1. Trên Storage Host (`vhhl1c2lab2com05`)

Đo trên file ảnh backstore của `disk1` (file này bản thân nó là một image thô):

  

Bash

```
# Target: đường dẫn file .img
# Label: storage-local
# Không cần offset vì file này chỉ là data thô của LUN
sudo ./bench-raw.sh /var/target/disk1.img storage-local
```

#### 2. Trên KVM Host (`vhhl1c2lab2com07`)

Đo trên thiết bị Multipath Device Mapper `/dev/mapper/mpatha`:

  

Bash

```
# Tắt VM trước: sudo virsh destroy test-vm-iscsi-boot
# Nếu muốn an toàn tuyệt đối cho VM:
sudo ./bench-raw.sh /dev/mapper/mpatha kvm-mpath 25G 20G

# (Hoặc nếu chấp nhận xóa VM để test từ sector 0):
sudo ./bench-raw.sh /dev/mapper/mpatha kvm-mpath
```

#### 3. Trong Guest VM (`test-vm-iscsi-boot`)

Đo trên ổ `/dev/vda` (đang chạy OS của VM):

Bash

```
# Bắt buộc phải có OFFSET=25G và SIZE=20G để tránh ghi đè vào OS:
sudo ./bench-raw.sh /dev/vda vm-guest 25G 20G
```