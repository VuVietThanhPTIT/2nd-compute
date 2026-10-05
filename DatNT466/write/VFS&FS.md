### 1. File System (FS) — Hệ thống tệp cụ thể

Thuật ngữ **FS** (Concrete File System) đại diện cho phương pháp và cấu trúc dữ liệu dùng để **tổ chức, lưu trữ và định vị byte dữ liệu** trên một thiết bị lưu trữ thực tế (hoặc bộ nhớ).

* **Bản chất:** Mỗi FS có một cấu trúc ghi đĩa (on-disk format) riêng biệt:
* **ext4** dùng khối block, block bitmap, inode table, extent tree.
* **FAT32** dùng bảng File Allocation Table (FAT) và cluster.
* **NTFS** dùng bảng Master File Table (MFT).
* **NFS** không lưu trên đĩa cục bộ mà chuyển đổi thao tác thành gói tin mạng RPC.


* **Vấn đề nếu chỉ có FS:** Mỗi FS có cách quản lý siêu dữ liệu (metadata), kích thước block và giới hạn file hoàn toàn khác nhau. Nếu hệ điều hành kết nối trực tiếp ứng dụng với từng FS, lập trình viên sẽ phải viết code riêng cho từng loại ổ đĩa: đọc file trên USB phải dùng API khác với đọc file trên SSD.

---

### 2. Virtual File System (VFS) — Hệ thống tệp "Ảo"

Từ **"Virtual" (Ảo)** ở đây không mang nghĩa máy ảo (VM) hay bộ nhớ ảo (Virtual Memory), mà mang nghĩa **ảo hóa giao diện (Interface Abstraction)**.

VFS là một lớp phần mềm trừu tượng nằm bên trong nhân hệ điều hành (kernel), đứng giữa các hàm gọi hệ thống (system call như `open()`, `read()`, `write()`) và các FS cụ thể bên dưới.

**Tại sao lại gọi là "Ảo"?**

1. **Nó không có dữ liệu thật trên đĩa:** Bản thân VFS không sở hữu thuật toán phân bổ block hay cấu trúc on-disk format nào. Nó tồn tại hoàn toàn trong RAM dưới dạng các cấu trúc dữ liệu của kernel khi hệ thống đang chạy.
2. **Tạo ra một ảo ảnh đồng nhất (Single System Image):** VFS "lừa" các ứng dụng tầng user-space tin rằng toàn bộ hệ thống tệp chỉ là một cây phân cấp duy nhất (`/`), có hành vi đồng nhất. Dù một thư mục con nằm trên ổ NVMe (ext4), một thư mục khác nằm trên USB (FAT32), và một thư mục khác kéo từ server qua mạng (NFS), ứng dụng chỉ cần gọi đúng một hàm `read(fd, buf, count)`.
3. **Hoạt động như một Object-Oriented Interface trong ngôn ngữ C:**
VFS định nghĩa ra 4 đối tượng trừu tượng chính:
* `superblock`: Đại diện cho toàn bộ hệ thống tệp đã mount.
* `inode`: Đại diện cho một file hoặc thư mục cụ thể (metadata, quyền hạn).
* `dentry` (directory entry): Quản lý quan hệ đường dẫn tên file và inode (hỗ trợ cache đường dẫn).
* `file`: Đại diện cho một file đang được một tiến trình mở (lưu con trỏ đọc/ghi `offset`, cờ chế độ).


Mỗi đối tượng này chứa các bảng con trỏ hàm (`inode_operations`, `file_operations`). Khi ứng dụng gọi `write()`, VFS sẽ tra cứu bảng này và điều hướng lời gọi tới driver của FS cụ thể tương ứng (ví dụ: `ext4_file_write_iter`).

---

### So sánh trực tiếp giữa VFS và FS

| Tiêu chí | Virtual File System (VFS) | Concrete File System (FS) |
| --- | --- | --- |
| **Vị trí** | Tầng trừu tượng phía trên trong kernel | Tầng triển khai cụ thể phía dưới |
| **Không gian tồn tại** | Chỉ trong bộ nhớ RAM của kernel | Ghi trực tiếp lên đĩa vật lý / bộ nhớ / mạng |
| **Cấu trúc on-disk** | Không có | Có định dạng đĩa cố định (superblock, inode table...) |
| **Mục đích** | Thống nhất API, đa hình hóa thao tác file | Lưu trữ bền vững, tối ưu I/O cho phần cứng |
| **Ví dụ** | Lớp VFS của Linux | ext4, XFS, Btrfs, FAT32, NTFS, NFS |

---

### Những ngộ nhận và chi tiết kỹ thuật ít được nhắc tới

* **Nhầm lẫn giữa VFS và Pseudo-FS:** Nhiều người hay nhầm VFS với các hệ thống tệp ảo như `/proc`, `/sys` hay `tmpfs`.
* *Sự thật:* `/proc` và `/sys` là các **Pseudo-Filesystem** (hệ thống tệp giả định sinh dữ liệu động trên RAM). Chúng là các **FS cụ thể**, nhưng không ghi dữ liệu xuống đĩa.
* VFS là **bộ khung điều phối** mà cả ext4 lẫn `/proc` đều phải đăng ký vào để hoạt động.


* **Nguồn gốc lịch sử:** VFS được Sun Microsystems giới thiệu lần đầu vào năm 1985 khi họ phát triển giao thức NFS cho hệ điều hành SunOS. Trước thời điểm đó, Unix bị gắn chặt cứng nhắc với kiến trúc UFS (Unix File System), khiến việc tích hợp mạng hay định dạng file mới bắt buộc phải can thiệp phá vỡ kernel.
* **Cái giá của lớp trừu tượng:** Tính linh hoạt của VFS đi kèm chi phí: kernel phải tốn thêm các bước tra cứu bảng con trỏ hàm gián tiếp (function pointer indirection), chuyển đổi qua lại giữa đối tượng inode của VFS và cấu trúc nội bộ của từng FS cụ thể, cũng như phải duy trì bộ nhớ đệm cấu trúc đường dẫn (Dentry Cache) trong RAM.

