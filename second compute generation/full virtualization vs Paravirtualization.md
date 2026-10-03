Thay vì mô phỏng hoàn toàn ảo hóa , từng lệnh là từng exit xuống host , thì para virt xử lý bằng cách là dùng hugepages , share memory , hoặc polling 

  

### 1. Phân loại theo cơ chế tương tác với phần cứng (Hardware / Hypervisor level)

Ngoài Full và Para, còn có các dạng mở rộng rất quan trọng:

  

- **Full Virtualization (Ảo hóa toàn phần):**  ( EX : qemu )
    - Hypervisor giả lập hoàn chỉnh toàn bộ kiến trúc phần cứng.  
    - Guest OS không cần chỉnh sửa bất kỳ dòng mã nào, chạy như trên máy vật lý thật.    
    - _Thời kỳ đầu:_ Sử dụng kỹ thuật Binary Translation (dịch nhị phân) rất chậm.
    
- **Hardware-Assisted Virtualization (Ảo hóa có phần cứng hỗ trợ):**
    
    - Thực chất là một bước tiến hóa của Full Virtualization nhờ CPU hỗ trợ trực tiếp (Intel VT-x, AMD-V) với các tập lệnh chuyển đổi chế độ VMX Root / Non-Root.
    - KVM chính là ví dụ điển hình của loại này: CPU chạy trực tiếp mã lệnh của Guest mà không cần Hypervisor dịch nhị phân.

- **Paravirtualization (Ảo hóa một phần / Giả ảo hóa):**
    - Guest OS được sửa đổi hoặc cài các driver chuyên dụng (như VirtIO, Xen Hypercalls) để chủ động nhận biết mình đang trong máy ảo và giao tiếp trực tiếp với Hypervisor qua các lệnh gọi đặc biệt thay vì trap lỗi phần cứng.
           
- **Hybrid Virtualization (Ảo hóa lai):**
  
    - **Phổ biến nhất trong thực tế hiện nay (chính là mô hình KVM + QEMU + VirtIO): CPU và RAM dùng _Hardware-Assisted Virtualization_, còn I/O (ổ đĩa, mạng) thì dùng _Paravirtualization_ (VirtIO) để đạt hiệu năng tối đa.**
### 2. Phân loại theo tầng kiến trúc (Levels of Virtualization)

Nếu xét rộng ra toàn bộ bức tranh ảo hóa:


1. **Hardware / System Virtualization (Ảo hóa phần cứng):** Tạo ra cả máy ảo hoàn chỉnh (gồm Full, Para, Hardware-assisted như nói ở trên - KVM, VMware ESXi, Xen, Proxmox).
2. **OS-Level Virtualization (Ảo hóa cấp hệ điều hành):** Không tạo ra máy ảo hoàn chỉnh, chia sẻ chung một Kernel của Host. Đây chính là **Container** (Docker, LXC, Podman, cgroups & namespaces).
3. **Application / Process Virtualization:** Ảo hóa môi trường chạy cho một ứng dụng cụ thể (ví dụ: Java Virtual Machine - JVM, Wine để chạy file `.exe` trên Linux).
4. **Desktop / Storage / Network Virtualization:** Ảo hóa tài nguyên chuyên biệt như VDI, SAN/Ceph ảo hóa lưu trữ, SDN/Open vSwitch ảo hóa mạng.

### Nhanh nhất là qemu + kvm + virtIO > qemu + kvm 


----

![[Pasted image 20261002165516.png]]