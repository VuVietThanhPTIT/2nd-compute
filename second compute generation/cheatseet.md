Dưới đây là cheatsheet các lệnh `iscsiadm` đã gắn sẵn thông số của bạn (`10.11.4.21:3260` và `iqn.2026-09.local.fptcloud:storage.target1`) để copy và chạy trực tiếp trên **KVM Host (`com07`)**:

  

### 1. Kiểm tra trạng thái hiện tại

Bash

```
# Xem các session đang kết nối
sudo iscsiadm -m session -P 1

# Xem danh sách các target đã lưu trong cache
sudo iscsiadm -m node
```

### 2. Đăng xuất (Logout) và Xóa sạch Session / Cache cũ

Bash

```
# 1. Đăng xuất khỏi target cụ thể
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 -u

# (Hoặc) Đăng xuất TẤT CẢ các session đang chạy
sudo iscsiadm -m node -u

# 2. Xóa sạch các bản ghi node cũ bị lưu sai trong database của iscsiadm
sudo iscsiadm -m node -o delete

# 3. Xóa cache discovery
sudo iscsiadm -m discoverydb -o delete
```

### 3. Khởi động lại dịch vụ iSCSI & Multipath

Bash

```
# Khởi động lại daemon iscsid và multipath
sudo systemctl restart iscsid open-iscsi multipathd

# Kiểm tra trạng thái service
sudo systemctl status iscsid --no-pager
```

### 4. Dò tìm (Discovery) và Đăng nhập (Login) mới

Bash

```
# 1. Quét tìm target từ Storage (Discovery)
sudo iscsiadm -m discovery -t sendtargets -p 10.11.4.21:3260

# 2. Đăng nhập vào target đích danh
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 -l

# (Hoặc) Đăng nhập TẤT CẢ các target vừa tìm thấy
sudo iscsiadm -m node -l

# 3. Đặt tự động đăng nhập lại khi boot máy
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 --op update -n node.startup -v automatic
```

### 5. Quét nhận diện đĩa & Multipath sau khi Login

Bash

```
# Quét lại session iSCSI để nhận LUN mới
sudo iscsiadm -m session --rescan

# Quét lại toàn bộ SCSI bus trên host
sudo rescan-scsi-bus.sh

# Cập nhật lại device-mapper multipath
sudo multipath -r

# Kiểm tra ổ đĩa đã nhận chưa
lsblk
sudo multipath -ll
```

### ⚡ Combo 1 lần chạy: Xóa sạch $\to$ Restart $\to$ Login lại từ đầu

Nếu bị treo hoặc lỗi không rõ nguyên nhân, dán cụm lệnh này để reset toàn bộ kết nối:

Bash

```
sudo iscsiadm -m node -u
sudo iscsiadm -m node -o delete
sudo systemctl restart iscsid
sudo iscsiadm -m discovery -t sendtargets -p 10.11.4.21:3260
sudo iscsiadm -m node -T iqn.2026-09.local.fptcloud:storage.target1 -p 10.11.4.21:3260 -l
sudo multipath -r
lsblk
```