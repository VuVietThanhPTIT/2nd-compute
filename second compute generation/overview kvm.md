
### 1. Triết lý thiết kế & Vị thế thực tế

- **Triết lý cốt lõi:** _"Không cần tạo ra hypervisor mới, chính Linux là hypervisor"_. Thay vì viết lại hàng triệu dòng code như VMware ESXi, KVM chỉ bổ sung khoảng 10.000 dòng code hạt nhân để tận dụng toàn bộ bộ lập lịch (CFS Scheduler), quản lý bộ nhớ, bảo mật (SELinux) và driver phần cứng sẵn có của Linux.
- **Vị thế:** Nền tảng hạ tầng của Google Cloud (GCE), AWS EC2 (qua kiến trúc Nitro), hơn 45 triệu core OpenStack và là đích đến của làn sóng chuyển dịch từ VMware.
- **Phân loại Hypervisor:** KVM là **Hybrid Type 1** (chạy thẳng trên phần cứng bare-metal vì Linux Kernel lúc này đóng vai trò chính là hypervisor, cho hiệu năng đạt 95–98% so với máy thật).
### 2. Kiến trúc 3 tầng và cơ chế CPU/Memory

- **Tầng phần cứng (Hardware Extensions):**  
    - Dùng tập lệnh ảo hóa Intel VT-x (`VMXON`, `VMLAUNCH`, `VMRESUME`, `VMEXIT`) hoặc AMD-V (`VMRUN`).  
    - Cấu trúc dữ liệu điều khiển cho mỗi vCPU nằm trong **VMCS** (Intel) hoặc **VMCB** (AMD).
- **Tầng Kernel (`kvm.ko`):**  
    - Chia chế độ CPU thành **VMX Root** (KVM chạy) và **VMX Non-root** (Guest chạy). Hai chế độ này vuông góc độc lập với Ring 0/Ring 3. 
    - Mỗi khi Guest thực thi lệnh nhạy cảm hoặc truy cập thiết bị, CPU sẽ kích hoạt **VM-Exit** (tiêu tốn từ 1.000–10.000 chu kỳ CPU) để chuyển quyền kiểm soát về KVM xử lý rồi quay lại bằng **VM-Entry**. 
- **Tầng Userspace (QEMU, Firecracker, crosvm):**  
    - Giao tiếp với Kernel qua thiết bị ký tự `/dev/kvm` bằng các lệnh `ioctl(KVM_CREATE_VM)`, `ioctl(KVM_CREATE_VCPU)` và vòng lặp `ioctl(KVM_RUN)`.    
    - **Mỗi vCPU thực chất là một thread POSIX** của tiến trình QEMU, chịu sự phân bổ tài nguyên bởi Linux Scheduler.
- **Ảo hóa bộ nhớ (Memory Virtualization):**
    - Dùng công nghệ phần cứng **EPT (Intel) / NPT (AMD)** để CPU tự động dịch địa chỉ hai lớp ($\text{GVA} \rightarrow \text{GPA} \rightarrow \text{HPA}$) mà không gây VM-Exit (ngoại trừ khi có page fault / EPT violation).
### 3. Ba kỹ thuật ảo hóa I/O

1. **Full Emulation:** QEMU giả lập hoàn toàn phần cứng cổ điển (e1000, IDE). Bị thắt cổ chai hiệu năng do mỗi lệnh I/O đều gây VM-Exit chuyển đổi ngữ cảnh liên tục. 
2. **VirtIO Paravirtualization:** Dùng vòng nhớ đệm chia sẻ (**Virtqueues**) trên RAM giữa Guest và Host. Guest chỉ ghi vào thanh ghi một lần (Doorbell kick) để Host xử lý hàng loạt (batching), giảm thiểu số lần VM-Exit:  
    - _vhost-net:_ Đưa Datapath vào Kernel của Host để tránh chuyển đổi qua QEMU (~10 Gbps).  
    - _vhost-user (DPDK):_ Chạy cơ chế Polling Mode hoàn toàn trong userspace (~40–100 Gbps).
3. **SR-IOV & PCI/GPU Passthrough:** Chia card mạng vật lý thành các Virtual Functions (VF) hoặc bàn giao trực tiếp toàn bộ GPU vào máy ảo qua IOMMU/VFIO. Đạt hiệu năng 100% bare-metal, độ trễ cực thấp nhưng đánh đổi bằng việc không thể Live Migration theo cách thông thường.
### 4. Cơ chế di trú sống (Live Migration)
- **Quy trình 4 pha:** 
    1. _Setup:_ Thiết lập kết nối TCP giữa host nguồn và đích, kiểm tra độ tương thích CPU.
    2. _Pre-copy:_ Vòng lặp truyền bộ nhớ RAM qua mạng trong khi VM vẫn chạy; các vòng sau chỉ truyền các trang bị ghi đè (dirty pages).
    3. _Stop-and-Copy:_ Dừng vCPU (thời gian dừng chỉ khoảng 50–200ms) để gửi nốt những trang nhớ dirty cuối cùng cùng trạng thái CPU trong VMCS.
    4. _Switchover:_ Đích bật chạy vCPU, phát bản tin ARP thông báo địa chỉ IP mới và giải phóng VM ở nguồn.
- **Xử lý ứng dụng nặng (Memory Write-Intensive):** Nếu tốc độ dirty page nhanh hơn băng thông mạng, KVM áp dụng kỹ thuật bóp CPU máy ảo (CPU Throttling) hoặc chuyển sang mô hình **Post-copy** (chuyển vCPU sang đích chạy ngay lập tức rồi kéo các trang nhớ còn thiếu về qua mạng sau).
- **NET_FAILOVER:** Cho phép máy ảo dùng SR-IOV có thể Live Migration bằng cách tạo cặp thiết bị dự phòng trong Guest (SR-IOV chính + VirtIO phụ). Khi migrate, ngắt SR-IOV chuyển sang VirtIO, sau khi sang host mới thì cắm lại SR-IOV.
### 5. Tinh chỉnh hiệu năng chuẩn Production (Tuning)
- **CPU:** Sử dụng **CPU Pinning** (`<vcpupin>`) để ghim vCPU vào core vật lý cố định tránh mất L1/L2 cache; tối ưu cấu trúc **NUMA Node** để RAM và vCPU nằm cùng một socket phần cứng; chọn `-cpu host` để tận dụng toàn bộ tập lệnh CPU gốc.
- **Memory:** Sử dụng **Huge Pages (1GB / 2MB)** để giảm số lượng bảng trang và giảm TLB miss (tăng 20% hiệu năng); dùng **Memory Ballooning** hoặc KSM (Kernel Samepage Merging) khi cần overcommit tài nguyên.
- **Storage & Network:** Cấu hình `cache=none` kết hợp `aio=native` hoặc `io_uring`; tách riêng `iothread` cho từng thiết bị đĩa; bật `vhost-net` và chế độ Multi-queue cho card mạng virtio.
### 6. Bảo mật & KVM trong kỷ nguyên AI
- **Bảo mật phân tầng:** Cách ly phần cứng qua EPT/IOMMU; bảo mật tiến trình qua **sVirt** (SELinux gán nhãn riêng cho từng tiến trình VM và file ảnh đĩa).
- **Confidential Computing:** Công nghệ **AMD SEV-SNP** và **Intel TDX** giúp mã hóa toàn bộ dữ liệu trên RAM và thanh ghi CPU bằng phần cứng, đảm bảo ngay cả quản trị viên máy chủ vật lý hay nhà cung cấp điện toán đám mây cũng không thể xem hoặc can thiệp vào bộ nhớ máy ảo.
- **AI & GPU Workloads:**
    - Hỗ trợ NVIDIA vGPU và kỹ thuật chia phần cứng **MIG (Multi-Instance GPU)**.
    - Tích hợp **Firecracker MicroVM** (mở máy chỉ mất ~125ms so với 3s của QEMU) để chạy tác vụ Serverless AI Inference quy mô lớn.  
    - Xu hướng **Confidential AI**: Bảo vệ trọng số mô hình AI (Model Weights) và dữ liệu huấn luyện y tế/tài chính không bị nhà cung cấp hạ tầng cloud đọc trộm.