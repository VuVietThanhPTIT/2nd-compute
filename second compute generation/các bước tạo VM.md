

1. **Khai biến và proxy.** Mở terminal mới thì phải khai lại.
    
    ```bash
    export IMG=/var/lib/libvirt/images/bench
    export PROXY=http://10.10.2.22:3001
    export HTTP_PROXY=$PROXY HTTPS_PROXY=$PROXY http_proxy=$PROXY https_proxy=$PROXY
    ```

    ```bash
    apt-get update && apt-get install -y qemu-kvm libvirt-daemon-system virtinst cloud-image-utils wget
    virsh net-start default 2>/dev/null || true
    ```



    
5. **Tạo thư mục và có base image** Ubuntu 22.04 (tải hoặc copy từ `io-lab-vm01`):
    
    ```bash
    mkdir -p $IMG
    cp /var/lib/libvirt/images/io-lab-vm01/ubuntu-22.04.qcow2 $IMG/
    qemu-img info $IMG/ubuntu-22.04.qcow2
    ```
    
6. **Tạo đĩa OS 20G** (overlay lên base, nên base phải giữ nguyên chỗ):
    
    ```bash
    qemu-img create -f qcow2 -F qcow2 -b $IMG/ubuntu-22.04.qcow2 $IMG/vm-bench-iscsi-root.qcow2 20G
    ```
    
7. **Tạo seed.iso cloud-init** (user `ubuntu`/`123456`, proxy cho apt, tự cài `fio`, `jq`). `EOF` không có nháy đơn để `$PROXY` được thay giá trị:
    
    ```bash
    cat > /tmp/user-data << EOF
    #cloud-config
    users:
      - name: ubuntu
        plain_text_passwd: '123456'
        lock_passwd: false
        sudo: ALL=(ALL) NOPASSWD:ALL
        shell: /bin/bash
    ssh_pwauth: true
    apt:
      http_proxy: $PROXY
      https_proxy: $PROXY
    package_update: true
    packages:
      - fio
      - jq
      - bsdextrautils
    EOF
    cloud-localds $IMG/vm-bench-iscsi-seed.iso /tmp/user-data
    ```
    
8. **Tạo VM** với 3 đĩa: OS (`vda`), `mpathb` (`vdb`, đĩa test, bắt buộc `raw`), CD cloud-init. Bước này gắn đĩa thật vào VM nên test ghi về sau sẽ phá dữ liệu trên đó.
    
    ```bash
    virt-install \
      --name vm-bench-iscsi --memory 4096 --vcpus 4 --os-variant generic \
      --disk path=$IMG/vm-bench-iscsi-root.qcow2,format=qcow2,bus=virtio \
      --disk path=/dev/mapper/mpathb,format=raw,bus=virtio,cache=none,io=native \
      --disk path=$IMG/vm-bench-iscsi-seed.iso,device=cdrom \
      --network network=default,model=virtio \
      --graphics none --serial pty --console pty --import --noautoconsole
    ```
    
9. **Vào VM và kiểm tra.** Đợi khoảng 1 phút:
    
    ```bash
    virsh domifaddr vm-bench-iscsi         # lấy IP
    ssh ubuntu@<IP>                        # mật khẩu 123456
    ```
	    
    Trong VM:
    
    ```bash
    cloud-init status --wait               # phải ra "status: done"
    command -v fio jq column
    lsblk                                  # vda ~20G, vdb ~447G
    ```
    

Xong là VM sẵn sàng để chạy `bench.sh` trên `/dev/vdb`.

**Sau khi đo xong, dọn VM** (không đụng `io-lab-vm01` hay `mpatha`):

```bash
virsh destroy vm-bench-iscsi
virsh undefine vm-bench-iscsi
rm -f $IMG/vm-bench-iscsi-root.qcow2 $IMG/vm-bench-iscsi-seed.iso
```

Ba lỗi hay gặp khi làm lại:

- Quên `export IMG` ở terminal mới, nên các lệnh trỏ vào `/ubuntu-22.04.qcow2` và báo không thấy file.
- `-b` trỏ vào một thư mục (như `-b .`) hoặc đường dẫn sai, nên `qemu-img` báo không mở được backing file.
- Gắn nhầm `mpatha` 20G thay vì `mpathb`: luôn xem `lsblk` ở bước 2 trước.

Dưới đây là từng bước riêng, chạy lần lượt và xem kết quả mỗi bước trước khi sang bước sau. Mở terminal mới thì phải khai lại biến ở Bước 0.

## Bước 0: Khai biến

```bash
export IMG=/var/lib/libvirt/images/bench
export PROXY=http://10.10.2.22:3001
export HTTP_PROXY=$PROXY HTTPS_PROXY=$PROXY http_proxy=$PROXY https_proxy=$PROXY
cd $IMG
```

## Bước 1: Kiểm tra hệ thống (dừng lại nếu có lỗi)

```bash
systemctl is-active dbus libvirtd systemd-machined     # phải ra 3 dòng active
timeout 40 systemd-run --scope --quiet true; echo "scope exit=$?"   # phải exit=0
ls -l /dev/kvm
virsh net-start default 2>/dev/null || true
virsh net-list                                          # default phải active
```

Nếu `scope exit` khác 0 hoặc `systemctl` báo lỗi thì dừng, lỗi cũ đã quay lại.

## Bước 2: Kiểm tra đĩa test `mpathb`

```bash
multipath -ll mpathb                                    # path phải active ready
lsblk /dev/mapper/mpathb                                # phải ~447G, không phải mpatha 20G
timeout 10 dd if=/dev/mapper/mpathb of=/dev/null bs=1M count=1 iflag=direct; echo "dd exit=$?"
virsh domblklist io-lab-vm01                            # không được có mpathb
```

## Bước 3: Dọn bản cũ (domain và file)

Domain `vm-bench-iscsi` đã `define` từ lần trước (nếu còn) cần xóa để tạo lại sạch:

```bash
virsh list --all
virsh destroy vm-bench-iscsi 2>/dev/null
virsh undefine vm-bench-iscsi 2>/dev/null
rm -f $IMG/vm-bench-iscsi-root.qcow2 $IMG/vm-bench-iscsi-seed.iso /tmp/vm-bench-iscsi.xml
```

## Bước 4: Kiểm tra base image

```bash
ls -lh $IMG/ubuntu-22.04.qcow2
qemu-img info $IMG/ubuntu-22.04.qcow2
```

Nếu thiếu file thì copy lại: `cp /var/lib/libvirt/images/io-lab-vm01/ubuntu-22.04.qcow2 $IMG/`

## Bước 5: Tạo đĩa OS 20G (overlay lên base)

```bash
qemu-img create -f qcow2 -F qcow2 -b $IMG/ubuntu-22.04.qcow2 $IMG/vm-bench-iscsi-root.qcow2 20G
qemu-img info --backing-chain $IMG/vm-bench-iscsi-root.qcow2 | grep -E 'image:|backing file:|virtual size'
```

## Bước 6: Tạo seed.iso cloud-init

`EOF` không có nháy đơn để `$PROXY` được thay giá trị:

```bash
cat > /tmp/user-data << EOF
#cloud-config
users:
  - name: ubuntu
    plain_text_passwd: '123456'
    lock_passwd: false
    sudo: ALL=(ALL) NOPASSWD:ALL
    shell: /bin/bash
ssh_pwauth: true
apt:
  http_proxy: $PROXY
  https_proxy: $PROXY
package_update: true
packages:
  - fio
  - jq
  - bsdextrautils
EOF

cloud-localds $IMG/vm-bench-iscsi-seed.iso /tmp/user-data
file $IMG/vm-bench-iscsi-seed.iso                       # phải ra "ISO 9660 ... 'cidata'"
grep -A2 '^apt:' /tmp/user-data                         # kiểm tra proxy đã được thay giá trị
```

## Bước 7: Sinh XML (chưa tạo VM)

```bash
rm -f /tmp/vm-bench-iscsi.xml
virt-install \
  --name vm-bench-iscsi --memory 4096 --vcpus 4 --os-variant ubuntu22.04 \
  --disk path=$IMG/vm-bench-iscsi-root.qcow2,format=qcow2,bus=virtio \
  --disk path=/dev/mapper/mpathb,format=raw,bus=virtio,cache=none,io=native \
  --disk path=$IMG/vm-bench-iscsi-seed.iso,device=cdrom \
  --network network=default,model=virtio \
  --graphics none --serial pty --console pty \
  --import --noautoconsole --print-xml > /tmp/vm-bench-iscsi.xml

ls -l /tmp/vm-bench-iscsi.xml                           # phải > 0 byte, nếu 0 byte thì KHÔNG define
```

## Bước 8: Define domain

```bash
virsh define /tmp/vm-bench-iscsi.xml
virsh list --all                                        # vm-bench-iscsi ở trạng thái shut off
```

## Bước 9: Start VM (có timeout, đừng Ctrl+C)

```bash
timeout 90 virsh start vm-bench-iscsi; echo "start exit=$?"
virsh list --all
```

Nếu `exit=124` hoặc báo lỗi, gửi output:

```bash
journalctl -u libvirtd -u systemd-machined --since "5 min ago" --no-pager | tail -30
```

## Bước 10: Vào VM và kiểm tra (đợi khoảng 1 phút)

```bash
virsh domifaddr vm-bench-iscsi                          # lấy IP
ssh ubuntu@<IP>                                         # mật khẩu 123456
```

Trong VM:

```bash
cloud-init status --wait                                # phải ra "status: done"
command -v fio jq column
lsblk                                                   # vda ~20G, vdb ~447G
```

## Bước 11: Dọn sau khi đo xong

```bash
virsh destroy vm-bench-iscsi
virsh undefine vm-bench-iscsi
rm -f $IMG/vm-bench-iscsi-root.qcow2 $IMG/vm-bench-iscsi-seed.iso
```

Nhớ rằng `bench.sh` ghi thẳng lên `/dev/vdb` (tức `mpathb`) nên sẽ phá dữ liệu trên đó. Đảm bảo `mpathb` không chứa gì cần giữ.