

### là lớp trừu tượng chung cho các file system ,nhằm tạo ra 1  hệ thống file duy nhất cho hệ thống 
![[Pasted image 20261001100941.png]]
VFS - ( shim layer  )nằm giữa tầng syscall và các filesystem cụ thể. 
- Userspace gọi I/O chuẩn posix   ( ? e) thực hiện lời gọi hàm I/O chuẩn C. )
- Thư viện sinh ra system call  , (`sys_open`, `sys_read`...) gửi xuống Kernel.
- System Call Interface chuyển yêu cầu tới **VFS** 
- VFS xác định tệp đó thuộc mount point nào  và driver file system nào ( (ví dụ driver `ext4` cho ổ cứng cục bộ, hoặc driver `nfs` cho mạng) 
- Driver của file system tiếp tục giao tiếp với Device Driver bên dưới để thực hiện I/O vật lý lên đĩa (HDD, SSD, NVMe).



![[Pasted image 20261001103304.png]]


- Kiến trúc : 
	- File object : struct file là mối quan hệ giữa process với file
		- Mô tả phương thức để process tương tác với file 
		- Được tạo ra khi 1 process open 1 file 
		- Không được lưu trữ sẵn trên ổ cứng 
	- Dentry object : **hiển thị tổ chức của file** 
		- Mô tả cấu trúc file trong thư mục : dentry cho biết tên nào nằm trong thư mục nào và trỏ tới inode nào.
	- inode là bản thân file :
		-  Bản thân file (metadata, vị trí dữ liệu)
		- ![[Pasted image 20261001110449.png]]
	- Super block : chứa thông tin chung của 1 loại file system 
		- block size, tổng số block, số inode, trạng thái filesystem...
		- VD : FAT 
			- block size 512 byte ( 1 lần đọc và 1 lần ghi ) do vậy có 2 loại là size và size on disk 
			- max file 4 G
	![[Pasted image 20261001111629.png]]
	![[Pasted image 20261001111823.png]]