Tài liệu này là bằng sáng chế của Mỹ số **US 10,445,247 B2** (cấp ngày 15/10/2019) do kỹ sư **Bandan Souryakanta Das** phát minh và nộp bởi **Red Hat, Inc.**:
- **Tên bằng sáng chế:** _"Switching between single-level and two-level page table translations"_ (Chuyển đổi linh hoạt giữa cơ chế phân trang 1 tầng và 2 tầng trong ảo hóa). 
- **Bản chất sáng chế:** Một kỹ thuật cho phép **Hypervisor (KVM)** tự động chuyển đổi qua lại giữa hai cơ chế dịch địa chỉ ảo sang vật lý: **EPT (Phần cứng - 2 tầng)** và **Shadow Page Tables / SPT (Phần mềm - 1 tầng)** dựa trên tần suất **TLB Miss** và số lần **VM-Exit**.
### 1. Bối cảnh & Vấn đề kỹ thuật đặt ra

Để dịch từ địa chỉ ảo của ứng dụng trong máy ảo (**GVA**) sang địa chỉ RAM vật lý thật trên Host (**HPA**), lịch sử ảo hóa có 2 giải pháp đối nghịch nhau:
1. Shadow Page Tables - SPT (1 tầng - Phần mềm):
    
    - _Ưu điểm:_ Bảng SPT ánh xạ trực tiếp $\text{GVA} \rightarrow \text{HPA}$ (chỉ mất 1 tầng tra cứu). Nếu bị TLB Miss, thời gian phần cứng dò tìm trang rất nhanh.
    - _Nhược điểm:_ Hypervisor phải đặt chế độ chống ghi (write-protect) lên toàn bộ bảng trang của Guest. Cứ mỗi lần Guest OS cập nhật bảng trang, CPU sẽ bị **VM-Exit** để văng ra Hypervisor cập nhật lại SPT, gây sụt giảm nghiêm trọng hiệu năng với các tác vụ tạo/hủy tiến trình liên tục.    
2. Extended Page Tables - EPT (2 tầng - Phần cứng Intel/AMD):  
    - _Ưu điểm:_ Guest tự do sửa đổi bảng trang mà **không bị VM-Exit**.   
    - _Nhược điểm:_ Quá trình dịch phải đi qua 2 bước ($\text{GVA} \rightarrow \text{GPA} \rightarrow \text{HPA}$). Nếu xảy ra hiện tượng **TLB thrashing / TLB Miss** liên tục (với các ứng dụng ngốn nhiều bộ nhớ), CPU phải thực hiện quá trình dò cây 2D (_2D Page Walk_) tốn tới 16–24 chu kỳ truy cập RAM cho mỗi lần trượt, khiến tốc độ thực thi bị chậm lại.
       
> **Nghịch lý:** Có những loại tải công việc (workloads) chạy EPT lại chậm hơn SPT do bị dính quá nhiều TLB Miss.
> 
>   

### 2. Ý tưởng cốt lõi của sáng chế

Thay vì bắt máy ảo cố định dùng EPT hoặc SPT từ đầu đến cuối, Red Hat đưa ra thuật toán giám sát và **chuyển đổi động (Dynamic Switching)** giữa 2 chế độ này ngay trong lúc máy ảo đang chạy:

```
                     [ Chế độ EPT (2 tầng) ]
                               │
               (Nếu số lần TLB Miss > Miss Threshold)[cite: 24, 26, 32]
                               │
                               ▼
                     [ Chế độ SPT (1 tầng) ]
                               │
            (Nếu số lần VM-Exit / Sửa SPT > Access Threshold)[cite: 27, 32]
                               │
                               ▼
                     [ Quay lại chế độ EPT ][cite: 27, 32]
```

### 3. Chi tiết quy trình hoạt động (Qua các Flowchart)

#### Bước 1: Khi đang ở chế độ EPT $\rightarrow$ Chuyển sang SPT (Hình 2 & 5)

- Hypervisor nạp con trỏ EPT vào VMCS và bật cờ EPT.
- Hypervisor đặt một bộ đếm thời gian (**Preemption Timer**, ví dụ mỗi 10ms) rồi trao quyền chạy cho vCPU.    
- Hết 10ms, CPU thoát ra Hypervisor. Hypervisor đọc thanh ghi phần cứng đo lường hiệu năng (**Performance Counter**) để đếm số lần **TLB Miss** đã xảy ra.
    
- Nếu số lần TLB Miss vượt qua ngưỡng quy định (**Miss Threshold**, ví dụ $> 50$ lần/ms):  
    - Hypervisor nhận định máy ảo đang bị nghẽn do 2D Page Walk.
       
    - Nó lưu lại con trỏ EPT (để dùng sau này), tạo ra bảng trang bóng (SPT) từ bảng trang của Guest, rồi **xóa bit EPT để chuyển sang chế độ SPT**.
   
#### Bước 2: Khi đang ở chế độ SPT $\rightarrow$ Chuyển ngược lại EPT (Hình 3)

- Trong chế độ SPT, mỗi khi Guest OS muốn sửa đổi bảng trang, CPU bị bẫy VM-Exit về Hypervisor để đồng bộ vào SPT 
- Hypervisor tăng một bộ đếm gọi là **Shadow Counter** (đếm số lần can thiệp/sửa đổi SPT).
- Nếu **Shadow Counter** vượt quá ngưỡng truy cập (**Access Threshold**):

    - Hypervisor nhận thấy chi phí do bị VM-Exit quá nhiều đã vượt qua lợi ích của việc tra cứu 1 tầng    
    - Hypervisor xóa bảng SPT (Flush SPT), nạp lại con trỏ EPT vào VMCS và **kích hoạt lại chế độ EPT**.
#### Kỹ thuật hỗ trợ: Ghim vCPU (CPU Pinning - Hình 4)

- Để việc đếm số lần TLB Miss được chính xác tuyệt đối, sáng chế chỉ rõ: Hypervisor sẽ **ghim (pin) vCPU của máy ảo vào một Core CPU vật lý cố định**.
- Việc này đảm bảo các chỉ số đo TLB Miss trong thanh ghi phần cứng là của chính máy ảo đó chứ không bị lẫn với các tiến trình khác trên Host.
### 4. Tóm tắt giá trị kỹ thuật

Bằng sáng chế này giải quyết triệt để sự đánh đổi (trade-off) kinh điển trong kiến trúc ảo hóa bộ nhớ: **Tự động dùng SPT khi máy ảo bị nghẽn do đọc/truy cập RAM nhiều (nhiều TLB Miss), và tự động quay về EPT khi máy ảo liên tục thay đổi cấu trúc bảng trang (nhiều VM-Exit)**.