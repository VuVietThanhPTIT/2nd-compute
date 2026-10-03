![[Pasted image 20261003101458.png]]
![[Pasted image 20261003101702.png]]
![[Pasted image 20261003101638.png]]
![[Pasted image 20261003101642.png]]

## 1 , Luồng khởi tạo : 
- qemu điều khiển kvm thông qua /dev/kvm  bằng các syscall ioctl()
	- Khởi tạo hypervisor : qemu mở file /dev/kvm để nhận file descriptor trung tâm ( kvm_fd)
	- Tạo máy ảo : qemu gọi ioctl ( kvm_fd  , KVM_CREATE_VM ) ( ioctl là 1 syscall ) ,Kernel KVM cấp phát cấu trúc `struct kvm` lưuu trong RAM đại diện cho VM và trả về `vm_fd` cho QEMU
	- Cấp phát RAM : 
			- QEMU dùng mmap() ( thư viện xin cấp phát vùng nhớ , gần giống malloc()) để xin host HVA( host virtual address) tức là xin **1 dải bộ nhớ ảo VMA trên host**
		- QEMU gọi ioctl(vm_fd  , KVM_SET_USER_MEMORY_REGION ) để đang ký vùng nhớ với KVM của guest 
		- KVM lưu cấu trúc này và nạp vào bảng phân trang phần cứng **EPT** (Intel Extended Page Tables) hoặc **NPT** (AMD), ánh xạ địa chỉ vật lý ảo của Guest (GPA) sang địa chỉ vật lý thật của Host (HPA). sau đó CPU tự động ánh xạ GPA sang HPA
			- Khi mà truy cập vào 1 địa chỉ HPA mà chưa được ánh xạ trong bảng EPT , VM exit nhường cho KVM và  điền trang đó vào bảng 
				- **Hugepages sinh ra** để làm bảng EPT nhỏ đi   ,số mục ánh xạ ít đi , và tối ưu hóa bộ đệm TLB của CPU 
	- **Tạo và gắn vCPU:**
		- Với mỗi vCPU được cấu hình, QEMU gọi `ioctl(vm_fd, KVM_CREATE_VCPU, vcpu_id)` để KVM khởi tạo cấu trúc phần cứng (VMCS trên Intel hoặc VMCB trên AMD). KVM trả về `vcpu_fd`
		- QEMU ánh xạ thêm một vùng đệm chia sẻ gọi là `struct kvm_run` giữa kernel và userspace để trao đổi dữ liệu nhanh.
# 2. Mô hình tiến trình & Vòng lặp vCPU (Execution Loop)

- **Bản chất của vCPU:** Trong QEMU, mỗi vCPU được chạy dưới dạng **một luồng POSIX (pthread)** thông thường. Bộ lập lịch (Linux CFS Scheduler) trên Host toàn quyền điều phối các thread này lên các CPU Core vật lý giống hệt mọi tiến trình khác
- **Vòng lặp thực thi:** Trong mỗi pthread của vCPU, QEMU chạy một vòng lặp vô tận `ioctl(vcpu_fd, KVM_RUN)`:
- ```
  for (;;) {
    /* 1. Trao quyền thực thi cho KVM / Phần cứng */
    ioctl(vcpu_fd, KVM_RUN, 0);

    /* 2. Khi vCPU bị VM-Exit và KVM không tự xử lý được */
    switch (run->exit_reason) {
        case KVM_EXIT_IO:
            handle_io_port(...);       // QEMU xử lý I/O Port
            break;
        case KVM_EXIT_MMIO:
            handle_mmio(...);          // QEMU xử lý Memory Mapped I/O
            break;
        case KVM_EXIT_HLT:
            handle_halt(...);          // Guest rơi vào trạng thái nghỉ
            break;
        default:
            handle_other_exits(...);
            break;
    }
}
  ```
# 3. Cơ chế phần cứng: VM-Entry và VM-Exit
- VM-Entry (Host $\rightarrow$ Guest):
	- KVM nạp trạng thái máy ảo từ vùng nhớ **VMCS** (Virtual Machine Control Structure) và kích hoạt lệnh vi mã `VMLAUNCH` hoặc `VMRESUME`
	- CPU chuyển từ chế độ **VMX Root** sang **VMX Non-Root**. Guest OS thực thi trực tiếp các lệnh nhị phân trên CPU vật lý với tốc độ gốc mà không cần biên dịch lại hay thông qua giả lập.
- **VM-Exit (Guest $\rightarrow$ Host):**
	- Khi Guest thực hiện các thao tác nhạy cảm (như đọc/ghi thanh ghi điều khiển, ghi vào vùng nhớ phần cứng MMIO, đọc lệnh I/O port, hoặc lệnh `HLT`), CPU tự động chặn lại và bẫy về chế độ **VMX Root**, đồng thời lưu trạng thái Guest vào VMCS.
	- Quyền kiểm soát trả về cho KVM.
# 4. Phối hợp xử lý I/O giữa KVM và QEMU
- **KVM tự xử lý trong Kernel (Fast-path):**
	- Nếu đó là ngắt cục bộ hoặc sự kiện mà KVM đã có module trong nhân hỗ trợ (như cập nhật bảng phân trang EPT, Local APIC timer), KVM tự cập nhật và gọi ngay `VMRESUME` để Guest chạy tiếp. Việc này không cần đánh thức Userspace QEMU, giúp tiết kiệm chu kỳ CPU.
- **Chuyển tiếp lên QEMU (Slow-path):** 
	- Nếu Guest chạm vào thanh ghi của một thiết bị ảo (ví dụ gửi gói mạng qua VirtIO, đọc sector ổ đĩa), KVM không biết thiết bị đó vận hành ra sao.
	- KVM ghi lý do thoát (Exit Reason) và thông số vào vùng nhớ `kvm_run`, sau đó kết thúc lệnh `ioctl(KVM_RUN)`
	- QEMU thức dậy, đọc dữ liệu trong `kvm_run`, tìm đúng mô hình thiết bị giả lập (Device Model) để xử lý logic (ví dụ: bốc gói tin từ Virtqueue, gọi system call `write()` hoặc `io_submit()` đẩy dữ liệu ra file ảnh đĩa hay card mạng TAP trên Host).
	- Sau khi hoàn tất, QEMU tiếp tục vòng lặp, gọi lại `ioctl(KVM_RUN)` để KVM đưa vCPU trở lại hoạt động bình thường.
# 5 . Tối ưu hóa nâng cao: Cắt giảm chi phí giao tiếp QEMU - KVM

Vì việc chuyển ngữ cảnh từ KVM (Kernel) lên QEMU (Userspace) qua VM-Exit rất tốn kém (1.000–10.000 chu kỳ CPU), kiến trúc hiện đại áp dụng các cơ chế tối ưu:
- **VirtIO & Virtqueues:** Sử dụng cấu trúc hàng đợi vòng (VRings) trên RAM dùng chung giữa Guest và QEMU, giúp dồn dữ liệu gửi hàng loạt (batching) thay vì mỗi lệnh I/O đều gây VM-Exit. 
- **`ioeventfd` & `irqfd`:** KVM tự bắt sự kiện ghi chuông cửa (Doorbell MMIO) của VirtIO và báo qua Linux `eventfd` mà không cần thoát vòng lặp vCPU về QEMU. Ngược lại, QEMU có thể tiêm ngắt ảo (Virtual Interrupt) vào Guest qua `irqfd` thẳng từ luồng I/O độc lập.
- **vhost-kernel / vhost-user:** Bàn giao toàn bộ Datapath của VirtIO từ QEMU xuống thẳng Kernel (`vhost-net`, `vhost-scsi`) hoặc sang tiến trình Userspace độc lập dùng DPDK/SPDK, biến QEMU thuần túy thành bộ điều khiển cấu hình (Control Plane)