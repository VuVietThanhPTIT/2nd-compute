<!-- ORIGINAL 0 BEGIN -->
==> Vấn đề là kích thước bảng phân trang như nào cho tối ưu với workload của hệ thống --> k có con số tốt nhất cho mọi trường hợp -->tùy trường hợp sẽ có con số tốt nhất
==> HupePage(cho phép thay đôi kích thước linh hoạt)

Bộ nhớ ảo (Virtual Memory) là một lớp trừu tượng do Hệ điều hành (HĐH) kết hợp với Phần cứng (đặc biệt là bộ quản lý bộ nhớ MMU trên CPU) tạo ra, nhằm "đánh lừa" mỗi tiến trình (process) rằng: Nó đang sở hữu một dải bộ nhớ liền mạch, riêng biệt và khổng lồ.

<!-- ORIGINAL 0 END -->

<!-- ORIGINAL 1 BEGIN -->

## Tại sao lại gọi là "Ảo"?

<!-- ORIGINAL 1 END -->

<!-- ORIGINAL 2 BEGIN -->

Trong thực tế, thanh RAM vật lý của bạn là tài nguyên dùng chung cho hàng trăm tiến trình. Dữ liệu thực sự của một tiến trình có thể bị xé nhỏ, nằm rải rác ở nhiều vị trí khác nhau trên RAM, thậm chí bị đẩy tạm xuống ổ cứng (Swap/Pagefile).

Tuy nhiên, tiến trình không hề biết điều này. Tiến trình chỉ làm việc với các địa chỉ ảo (Virtual Address). Khi CPU thực thi một lệnh truy xuất bộ nhớ, MMU sẽ tự động dịch địa chỉ ảo đó thành địa chỉ vật lý (Physical Address) trên RAM.

<!-- ORIGINAL 2 END -->

<!-- ORIGINAL 3 BEGIN -->

```text
[Tiến trình A]  ---> Địa chỉ ảo (0x00001000) ---\
                                                 +--> [MMU (Chuyển đổi)] ---> [RAM vật lý (Vị trí 0x8F002000)]
[Tiến trình B]  ---> Địa chỉ ảo (0x00001000) ---/
```

<!-- ORIGINAL 3 END -->

<!-- ORIGINAL 4 BEGIN -->

Xét hệ thống 32-bit với không gian bộ nhớ ảo $4\text{ GiB}$ ($2^{32}\text{ bytes}$)
---\> không gian bộ nhớ ảo là sao?

Con số 32-bit đại diện cho độ rộng của con trỏ (pointer) hoặc thanh ghi địa chỉ trong CPU:Với 32 bit nhị phân, CPU có thể tạo ra tối đa $2^{32}$ tổ hợp trạng thái khác nhau.

Xét hệ thống 32-bit với không gian bộ nhớ ảo $4\text{ GiB}$ ($2^{32}\text{ bytes}$) và mỗi PTE chiếm $4\text{ bytes}$:Nếu trang có kích thước $512\text{ bytes}$ ($2^9\text{ bytes}$):

Số lượng trang = $\frac{2^{32}}{2^9} = 2^{23} = 8.388.608\text{ trang}$.

Dung lượng Bảng phân trang = $8.388.608 \times 4\text{ bytes} = 33.554.432\text{ bytes} \approx \mathbf{32\text{ MiB}}$.

**Hậu quả:** $32\text{ MiB}$ này là chi phí quản lý cho chỉ 1 tiến trình. Nếu hệ thống chạy 100 tiến trình đồng thời, bạn mất $3.2\text{ GiB}$ RAM chỉ để lưu metadata phân trang, chưa tính dung lượng cho chương trình thực thi.

So sánh với kích thước chuẩn $4\text{ KiB}$ ($2^{12}\text{ bytes}$):

Số lượng trang = $\frac{2^{32}}{2^{12}} = 2^{20} = 1.048.576\text{ trang}$.

Dung lượng Bảng phân trang = $1.048.576 \times 4\text{ bytes} = \mathbf{4\text{ MiB}}$ (giảm 8 lần).

<!-- ORIGINAL 4 END -->

<!-- ORIGINAL 5 BEGIN -->

## (Internal Fragmentation)

<!-- ORIGINAL 5 END -->

<!-- ORIGINAL 7 BEGIN -->

### và mỗi tiến trình sẽ đc cấp 1 page riêng à

<!-- ORIGINAL 7 END -->

<!-- ORIGINAL 8 BEGIN -->

Hệ điều hành cấp phát bộ nhớ theo đơn vị nguyên trang. Khi một tiến trình hoặc một cấu trúc dữ liệu cần bộ nhớ, HĐH không thể cấp phát "một nửa trang" hay "vài KiB lẻ", mà phải cấp phát tròn theo số lượng trang.
Phần bộ nhớ nằm trong trang đã cấp phát nhưng không được tiến trình sử dụng chính là phân mảnh nội vi. Phần thừa này bị khóa lại và các tiến trình khác hoàn toàn không thể chạm tới.
## TLB 
CPU sử dụng bộ đệm TLB (Translation Lookaside Buffer) để lưu trữ các ánh xạ địa chỉ gần đây nhằm tăng tốc truy xuất.

<!-- ORIGINAL 8 END -->

<!-- ORIGINAL 9 BEGIN -->

```text
CPU phát địa chỉ ảo (VA)
         │
         ▼
┌──────────────────┐
│   L1 TLB Check   │ ── Hit (1-2 cycles) ──► Có ngay Địa chỉ Vật lý (PA)
└────────┬─────────┘
         │ Miss
         ▼
┌──────────────────┐
│   L2 TLB (STLB)  │ ── Hit (~7-12 cycles) ─► Có ngay Địa chỉ Vật lý (PA)
└────────┬─────────┘
         │ Miss
         ▼
┌───────────────────────────────────────────────────────────┐
│ HARDWARE PAGE TABLE WALK (MMU duyệt 4 tầng qua DRAM)     │
│ - Đọc PGD, PUD, PMD, PTE                                  │
│ - Tốn hàng trăm chu kỳ CPU (~100 - 300 cycles)            │
│ - Đẩy cặp ánh xạ vừa tìm được vào lại TLB                 │
└───────────────────────────────────────────────────────────┘
```

<!-- ORIGINAL 9 END -->

<!-- ORIGINAL 11 BEGIN -->

# Phân tích sâu: Bảng phân trang nhiều cấp (Multi-level Page Table)

<!-- ORIGINAL 11 END -->

<!-- ORIGINAL 12 BEGIN -->

## Vấn đề của Bảng phân trang phẳng (Flat Page Table)

<!-- ORIGINAL 12 END -->

<!-- ORIGINAL 13 BEGIN -->

Nếu dùng bảng phân trang 1 cấp (phẳng) cho hệ thống 32-bit với kích thước trang $4\text{ KiB}$:

Số trang ảo = $\frac{2^{32}}{2^{12}} = 2^{20} = 1.048.576\text{ trang}$.

Mỗi trang cần 1 dòng (PTE) kích thước $4\text{ bytes}$ $\rightarrow$ Bảng phân trang tốn $4\text{ MiB}$ RAM.

**Điểm bất hợp lý:** Bảng phân trang phẳng đòi hỏi $4\text{ MiB}$ này phải liền mạch và luôn nằm trong RAM. Nhưng thực tế, hầu hết các tiến trình chỉ dùng một phần rất nhỏ không gian ảo (ví dụ: vài trăm KiB cho mã lệnh ở vùng nhớ thấp và một ít cho Stack ở vùng nhớ cao, khoảng giữa trống hoàn toàn). Việc phải lưu các dòng quản lý cho những vùng nhớ trống không bao giờ dùng tới gây lãng phí RAM nghiêm trọng.

<!-- ORIGINAL 13 END -->

<!-- ORIGINAL 14 BEGIN -->

## Giải pháp: Bảng phân trang 2 cấp (2-Level Page Table trên x86 32-bit)

<!-- ORIGINAL 14 END -->

<!-- ORIGINAL 15 BEGIN -->

Thay vì dùng 1 mảng lớn chứa $1.048.576$ dòng, HĐH chia mảng này thành cấu trúc cây 2 cấp:

**Cấp 1:** Thư mục trang (Page Directory - PD): Chứa 1.024 dòng (PDE). Mỗi dòng trỏ tới một Bảng trang Cấp 2.

**Cấp 2:** Bảng trang (Page Table - PT): Mỗi bảng chứa 1.024 dòng (PTE). Mỗi dòng trỏ tới 1 Khung trang vật lý $4\text{ KiB}$.

<!-- ORIGINAL 15 END -->

<!-- ORIGINAL 16 BEGIN -->

### Cách tách địa chỉ ảo 32-bit:

<!-- ORIGINAL 16 END -->

<!-- ORIGINAL 17 BEGIN -->

**Một địa chỉ ảo 32-bit sẽ được MMU chia làm 3 phần:** 

$$\text{Địa chỉ ảo 32-bit} = [\text{Directory Index: 10 bits}] + [\text{Table Index: 10 bits}] + [\text{Offset: 12 bits}]$$

10 bits Directory Index ($2^{10} = 1024$): Dùng để tìm vị trí trong Cấp 1 (Page Directory).

10 bits Table Index ($2^{10} = 1024$): Dùng để tìm vị trí trong Cấp 2 (Page Table).

12 bits Offset ($2^{12} = 4096\text{ bytes}$): Định vị chính xác byte dữ liệu bên trong Trang $4\text{ KiB}$.

<!-- ORIGINAL 17 END -->

<!-- ORIGINAL 18 BEGIN -->

```text
Địa chỉ ảo (32-bit)
+-------------------+-------------------+-----------------------+
|  Directory (10b)  |    Table (10b)    |      Offset (12b)     |
+---------+---------+---------+---------+-----------+-----------+
          |                   |                     |
          v                   |                     |
  [Page Directory]            |                     |
  +---------------+           |                     |
  |  PDE 0        |           |                     |
  |  PDE 1 --------+          |                     |
  |  ...          ||          v                     |
  +---------------+|[Page Table (Cấp 2)]             |
                   +->+---------------+             |
                      |  PTE 0        |             |
                      |  PTE 1 --------+            |
                      |  ...          ||            v
                      +---------------+|[Khung RAM vật lý 4 KiB]
                                       +->+-------------------+
                                          | Byte dữ liệu      |
                                          +-------------------+
```

<!-- ORIGINAL 18 END -->

<!-- ORIGINAL 19 BEGIN -->

### Tại sao Phân cấp lại tiết kiệm RAM vượt trội?

<!-- ORIGINAL 19 END -->

<!-- ORIGINAL 20 BEGIN -->

**Cốt lõi nằm ở cơ chế Cấp phát theo nhu cầu (Lazy Allocation):** 

Ban đầu, HĐH chỉ cần tạo duy nhất Cấp 1 (Page Directory). Dung lượng của Cấp 1 chỉ vỏn vẹn: $1024 \times 4\text{ bytes} = \mathbf{4\text{ KiB}}$.

Các Bảng trang Cấp 2 chỉ được khởi tạo khi và chỉ khi tiến trình thực sự ghi/đọc dữ liệu vào vùng không gian ảo tương ứng.

Nếu một vùng không gian ảo khổng lồ không được sử dụng, dòng ở Cấp 1 tương ứng sẽ được đánh dấu là Invalid (không hợp lệ) và HĐH không cần tốn 1 byte RAM nào để tạo Bảng trang Cấp 2 cho vùng đó.

<!-- ORIGINAL 20 END -->

<!-- ORIGINAL 21 BEGIN -->

### Bài toán so sánh thực tế:

<!-- ORIGINAL 21 END -->

<!-- ORIGINAL 22 BEGIN -->

Xét một tiến trình nhỏ chỉ sử dụng $1\text{ MiB}$ bộ nhớ ở vùng thấp và $1\text{ MiB}$ ở vùng cao (Stack):

**Bảng phân trang phẳng:** Tốn cố định $4\text{ MiB}$ RAM.

**Bảng phân trang 2 cấp:** 

1 Page Directory (Cấp 1) = $4\text{ KiB}$.

1 Page Table (Cấp 2) cho vùng thấp = $4\text{ KiB}$ (quản lý $1024 \times 4\text{ KiB} = 4\text{ MiB}$ không gian ảo).

1 Page Table (Cấp 2) cho vùng cao = $4\text{ KiB}$.

**Tổng chi phí RAM:** $4\text{ KiB} + 4\text{ KiB} + 4\text{ KiB} = \mathbf{12\text{ KiB}}$ (Tiết kiệm 99,7% dung lượng RAM so với dạng phẳng).

<!-- ORIGINAL 22 END -->

<!-- ORIGINAL 23 BEGIN -->

# Cấu trúc Bảng Phân Trang 4 Tầng trên x86-64 (4-Level Paging)

<!-- ORIGINAL 23 END -->

<!-- ORIGINAL 24 BEGIN -->

Không gian địa chỉ ảo 64-bit chuẩn (Canonical Virtual Address) sử dụng 48 bit thực tế (từ bit 0 đến bit 47, các bit từ 48 đến 63 là sign-extension). 48 bit này được chia nhỏ thành 4 chỉ mục 9-bit và 1 offset 12-bit:

**Plaintext**

<!-- ORIGINAL 24 END -->

<!-- ORIGINAL 25 BEGIN -->

```text
47       39 38       30 29       21 20       12 11          0
┌──────────┬───────────┬───────────┬───────────┬─────────────┐
│ PGD Index│ PUD Index │ PMD Index │ PTE Index │ Page Offset │
│ (9 bits) │ (9 bits)  │ (9 bits)  │ (9 bits)  │  (12 bits)  │
└──────────┴───────────┴───────────┴───────────┴─────────────┘
     │           │           │           │            │
     ▼           ▼           ▼           ▼            ▼
  CR3 -> PGD -> PUD  ───►   PMD  ───►   PTE   ───► Khung trang vật lý 4 KiB
```

<!-- ORIGINAL 25 END -->

<!-- ORIGINAL 26 BEGIN -->

**PGD (Page Global Directory / PML4):** Bits $\to$ $2^9 = 512$ mục (entries), mỗi mục 8 bytes $\to$ Bảng PGD chiếm đúng $512 \times 8 = 4096\text{ bytes}$ (1 khung trang).

**PUD (Page Upper Directory / PDPT):** Bits $\to$ 512 mục.

**PMD (Page Middle Directory / PD):** Bits $\to$ 512 mục.

**PTE (Page Table Entry / PT):** Bits $\to$ 512 mục. Mỗi PTE trỏ tới một khung trang vật lý 4 KiB (Host Physical Address).

**Offset:** Bits \[11:0\] $\to$ Vị trí byte cụ thể bên trong trang 4 KiB ($2^{12} = 4096$).

**Chi phí ẩn:** Để đọc/ghi đúng 1 byte dữ liệu từ một địa chỉ ảo chưa từng được nạp, phần cứng CPU/MMU phải thực hiện một chuỗi Page Table Walk: đọc DRAM liên tiếp 4 lần (PGD $\to$ PUD $\to$ PMD $\to$ PTE) trước khi chạm được vào dữ liệu thực tế.

<!-- ORIGINAL 26 END -->

<!-- ORIGINAL 27 BEGIN -->

# Extended Page Tables (EPT) — Ảo Hóa Bộ Nhớ Cấp Phần Cứng

<!-- ORIGINAL 27 END -->

<!-- ORIGINAL 28 BEGIN -->

Trong môi trường ảo hóa máy ảo (KVM, ESXi, Hyper-V), bài toán dịch địa chỉ trở nên phức tạp gấp đôi vì xuất hiện hai hệ điều hành đồng thời:

**Guest OS:** Nghĩ rằng mình đang quản lý bộ nhớ vật lý thật, tự tạo bảng phân trang của riêng nó:

$$\text{GVA (Guest Virtual Address)} \longrightarrow \text{GPA (Guest Physical Address)}$$

**Host OS (Hypervisor):** Quản lý thanh RAM thực tế của máy chủ vật lý, cấp phát một phần RAM đó cho Guest:

$$\text{GPA (Guest Physical Address)} \longrightarrow \text{HPA (Host Physical Address)}$$

<!-- ORIGINAL 28 END -->

<!-- ORIGINAL 29 BEGIN -->

## Lịch sử: Từ Shadow Page Tables (Phần mềm) sang EPT (Phần cứng)

<!-- ORIGINAL 29 END -->

<!-- ORIGINAL 30 BEGIN -->

### Shadow Page Tables (Thời kỳ đầu - Chậm chạp):

<!-- ORIGINAL 30 END -->

<!-- ORIGINAL 31 BEGIN -->

Hypervisor phải tự tay duy trì một bảng phân trang ngầm ("bóng ma") dịch thẳng từ $\text{GVA} \to \text{HPA}$. Mỗi khi Guest OS thay đổi bảng phân trang của nó (ghi đè một mục PTE), CPU bẫy ngắt qua VM-Exit. Hypervisor phải giật mình thức dậy ở Host Ring 0, phân tích cú ghi, cập nhật bảng shadow, rồi mới cho Guest chạy tiếp. Hàng triệu cú VM-Exit khiến hiệu năng I/O và memory sụt giảm nghiêm trọng.

<!-- ORIGINAL 31 END -->

<!-- ORIGINAL 32 BEGIN -->

### EPT (Intel EPT) / NPT (AMD Nested Page Tables / RVI):

<!-- ORIGINAL 32 END -->

<!-- ORIGINAL 33 BEGIN -->

Intel đưa công nghệ SLAT (Second Level Address Translation) trực tiếp vào silicon của CPU từ kiến trúc Nehalem (2008). CPU phần cứng lúc này hiểu được cả hai bảng phân trang:

Thanh ghi CR3 của Guest trỏ tới bảng phân trang của Guest (gPT).

Thanh ghi điều khiển phần cứng ảo hóa EPTP (EPT Pointer) trong cấu trúc VMCS trỏ tới bảng phân trang EPT của Host.

**Plaintext**

<!-- ORIGINAL 33 END -->

<!-- ORIGINAL 34 BEGIN -->

```text
                               LUỒNG DỊCH ĐỊA CHỈ HAI CHIỀU CỦA EPT
┌─────────────────────────────────┐
│ Guest Virtual Address (GVA)     │
└────────────────┬────────────────┘
                 │ (Tra cứu Guest Page Table - gPT)
                 ▼
┌─────────────────────────────────┐
│ Guest Physical Address (GPA)    │
└────────────────┬────────────────┘
                 │ (Tra cứu Extended Page Table - EPT qua EPTP)
                 ▼
┌─────────────────────────────────┐
│ Host Physical Address (HPA)     │
└─────────────────────────────────┘
```

<!-- ORIGINAL 34 END -->

<!-- ORIGINAL 35 BEGIN -->

## Cơn Ác Mộng: "Two-Dimensional Page Walk" (Duyệt Phân Trang 2 Chiều)

<!-- ORIGINAL 35 END -->

<!-- ORIGINAL 36 BEGIN -->

EPT giải phóng hoàn toàn các cú VM-Exit khi Guest cập nhật Page Table, nhưng nó lại sinh ra một chi phí phần cứng khủng khiếp khi TLB Miss:

Bản thân bảng phân trang của Guest (gPT) nằm trên RAM của Guest, tức là địa chỉ của các bảng này đều là GPA.

Mỗi lần MMU muốn đọc 1 cấp trong bảng phân trang của Guest, nó lại phải dùng toàn bộ 4 cấp của EPT để dịch GPA đó sang HPA\!

Công thức tính số lần truy cập bộ nhớ khi xảy ra TLB Miss hoàn toàn:

$$\text{Số lần đọc DRAM} = (N_{\text{Guest}} + 1) \times (N_{\text{Host}} + 1) - 1$$

Với cả Guest và Host đều dùng bảng phân trang 4 tầng ($N = 4$):

$$\text{Số lần đọc DRAM} = (4 + 1) \times (4 + 1) - 1 = 25 - 1 = \mathbf{24\text{ LẦN!}}$$

**Plaintext**

<!-- ORIGINAL 36 END -->

<!-- ORIGINAL 37 BEGIN -->

```text
Chi tiết 24 lần truy cập bộ nhớ khi MMU dịch 1 địa chỉ GVA -> HPA:
1. Đọc gPGD  cần: 4 lần tra cứu EPT + 1 lần đọc nội dung gPGD entry  = 5 lần
2. Đọc gPUD  cần: 4 lần tra cứu EPT + 1 lần đọc nội dung gPUD entry  = 5 lần
3. Đọc gPMD  cần: 4 lần tra cứu EPT + 1 lần đọc nội dung gPMD entry  = 5 lần
4. Đọc gPTE  cần: 4 lần tra cứu EPT + 1 lần đọc nội dung gPTE entry  = 5 lần
5. Lấy HPA cuối cùng: 4 lần tra cứu EPT                             = 4 lần
─────────────────────────────────────────────────────────────────────────
TỔNG CỘNG: 5 + 5 + 5 + 5 + 4 = 24 LẦN TRUY CẬP DRAM CHO 1 LẦN TLB MISS!
```

<!-- ORIGINAL 37 END -->

<!-- ORIGINAL 38 BEGIN -->

Một cú TLB Miss trong máy ảo có thể tiêu tốn từ 400 đến 1000 chu kỳ CPU. Đây chính là lý do các hệ thống ảo hóa chịu tải ngẫu nhiên lớn thường bị drop hiệu năng bộ nhớ nghiêm trọng.

<!-- ORIGINAL 38 END -->

<!-- ORIGINAL 39 BEGIN -->

# 4\. Hugepages — Lời Giải Triệt Tiêu Điểm Nghẽn Phân Trang

<!-- ORIGINAL 39 END -->

<!-- ORIGINAL 40 BEGIN -->

Hugepages là kỹ thuật tăng kích thước trang từ 4 KiB lên 2 MiB hoặc 1 GiB bằng cách cắt bớt các tầng trung gian trong cây phân trang.

**Plaintext**

<!-- ORIGINAL 40 END -->

<!-- ORIGINAL 41 BEGIN -->

```text
A. TRANG TRUYỀN THỐNG 4 KiB (Dùng đủ 4 tầng):
   PGD (9b) ──► PUD (9b) ──► PMD (9b) ──► PTE (9b) ──► Offset (12b = 4 KiB)

B. HUGEPAGE 2 MiB (Bỏ tầng PTE, bật bit Page Size 'PS=1' tại PMD):
   PGD (9b) ──► PUD (9b) ──► PMD (9b) ───────────────► Offset (21b = 2 MiB)
   (PMD trỏ thẳng vào khối RAM vật lý 2 MiB liên tục)

C. HUGEPAGE 1 GiB (Bỏ tầng PTE và PMD, bật bit 'PS=1' tại PUD):
   PGD (9b) ───────────────► PUD (9b) ───────────────► Offset (30b = 1 GiB)
   (PUD trỏ thẳng vào khối RAM vật lý 1 GiB liên tục)
```

<!-- ORIGINAL 41 END -->

<!-- ORIGINAL 42 BEGIN -->

## Tại sao Hugepages giải quyết triệt để vấn đề của EPT và TLB Miss?

<!-- ORIGINAL 42 END -->

<!-- ORIGINAL 43 BEGIN -->

### 1\. Mở Rộng "TLB Reach" Lên Gấp Hàng Nghìn Lần

<!-- ORIGINAL 43 END -->

<!-- ORIGINAL 44 BEGIN -->

**Giữ nguyên kiến trúc phần cứng với một L1 D-TLB gồm 64 entries:** 

**Với 4 KiB pages:** $\text{Reach} = 64 \times 4\text{ KiB} = \mathbf{256\text{ KiB}}$ (TLB Miss xảy ra liên tục).

**Với 2 MiB Hugepages:** $\text{Reach} = 64 \times 2\text{ MiB} = \mathbf{128\text{ MiB}}$ (Bao phủ toàn bộ L3 Cache của các CPU hiện đại nhất).

**Với 1 GiB Hugepages:** $\text{Reach} = 64 \times 1\text{ GiB} = \mathbf{64\text{ GiB}}$ (Chỉ cần 64 dòng TLB là đủ để bao trọn toàn bộ RAM của một máy ảo Database lớn $\to$ Tỷ lệ TLB Hit đạt xấp xỉ 100%).

<!-- ORIGINAL 44 END -->

<!-- ORIGINAL 45 BEGIN -->

### 2\. Cắt Giảm Cấp Số Nhân Chu Kỳ Two-Dimensional Page Walk

<!-- ORIGINAL 45 END -->

<!-- ORIGINAL 46 BEGIN -->

Khi Host cấp phát RAM cho máy ảo dưới dạng Hugepages (ví dụ 1 GiB hoặc 2 MiB):

Tầng EPT của Host không còn là 4 cấp nữa mà rút xuống chỉ còn 2 cấp (với 1 GiB) hoặc 3 cấp (với 2 MiB).

Số lần truy cập bộ nhớ khi xảy ra TLB Miss giảm từ 24 lần xuống chỉ còn khoảng 8 đến 10 lần, giảm hơn 60% độ trễ phần cứng.

<!-- ORIGINAL 46 END -->

<!-- ORIGINAL 47 BEGIN -->

## Phân Loại Triển Khai Hugepages trên Linux

<!-- ORIGINAL 47 END -->

<!-- ORIGINAL 48 BEGIN -->

### A. Static Hugepages (hugetlbfs)

<!-- ORIGINAL 48 END -->

<!-- ORIGINAL 49 BEGIN -->

**Cơ chế:** Quản trị viên chủ động cấu hình trước khi boot hoặc cấp phát tĩnh thông qua sysctl (ví dụ: vm.nr\_hugepages = 1024). Kernel sẽ dành riêng một vùng DRAM vật lý liên tục và khóa cứng lại.

**Đặc tính:** Vùng nhớ này không bao giờ bị swap ra đĩa, không bao giờ bị phân mảnh. Thường dùng cho các ứng dụng hiệu năng cao cực đoan: QEMU/KVM VM RAM (-mem-prealloc), cơ sở dữ liệu Oracle/PostgreSQL, DPDK/SPDK.

**Nhược điểm:** Cứng nhắc. Nếu cấu hình dư thừa, lượng RAM này sẽ bị bỏ phí và các tiến trình thông thường khác không thể chạm vào.

<!-- ORIGINAL 49 END -->

<!-- ORIGINAL 50 BEGIN -->

### B. Transparent Huge Pages (THP)

<!-- ORIGINAL 50 END -->

<!-- ORIGINAL 51 BEGIN -->

**Cơ chế:** Tính năng tự động hoàn toàn của Linux Kernel. Các tiến trình thông thường vẫn gọi hàm malloc() cấp phát bộ nhớ bình thường. Luồng ngầm của kernel (khugepaged) sẽ liên tục quét bộ nhớ; nếu phát hiện 512 trang 4 KiB nằm liên tiếp nhau, nó sẽ tự động gom (collapse) chúng lại thành một trang 2 MiB duy nhất.

**Đánh đổi & Rủi ro:** 

Khi RAM bị phân mảnh cao, thao tác gom trang của khugepaged buộc phải kích hoạt cơ chế Memory Compaction. Quá trình này khóa bộ nhớ và di dời các trang vật lý, gây ra hiện tượng Latency Spikes (Độ trễ tăng đột biến ngẫu nhiên).

Vì lý do này, hầu hết các hệ thống cơ sở dữ liệu chuyên nghiệp (Redis, MongoDB, Cassandra) và máy chủ ảo hóa KVM đều khuyến cáo vô hiệu hóa THP (`echo never > /sys/kernel/mm/transparent_hugepage/enabled`) và chuyển sang dùng Static Hugepages.

<!-- ORIGINAL 51 END -->

<!-- ORIGINAL 52 BEGIN -->

# 5\. Bảng So Sánh Toàn Diện Các Cơ Chế

<!-- ORIGINAL 52 END -->

<!-- ORIGINAL 53 BEGIN -->

| Thuộc tính Phân tích | Trang Truyền Thống (4 KiB) | Hugepage 2 MiB | Hugepage 1 GiB |
| --- | --- | --- | --- |
| Số tầng Page Table (x86-64) | 4 tầng (PGD $\to$ PUD $\to$ PMD $\to$ PTE) | 3 tầng (PGD $\to$ PUD $\to$ PMD) | 2 tầng (PGD $\to$ PUD) |
| Độ dài Offset địa chỉ | 12 bits | 21 bits | 30 bits |
| Số mục cần để ánh xạ 1 GiB RAM | 262,144 PTEs (~2 MiB metadata) | 512 PMDs (~4 KiB metadata) | 1 PUD (~8 bytes metadata) |
| Độ phủ L1 TLB (64 entries) | 256 KiB | 128 MiB | 64 GiB |
| Tần suất TLB Miss | Cực cao khi chạy tải ngẫu nhiên lớn | Rất thấp | Gần như bằng 0 |
| Độ trễ EPT Walk trong VM | Rất nặng (~24 lần đọc DRAM) | Trung bình (~15 lần đọc DRAM) | Thấp nhất (~8 lần đọc DRAM) |
| Nguy cơ Phân mảnh Nội vi | Thấp nhất | Trung bình | Rất cao (lãng phí RAM nếu không dùng hết) |
| Cơ chế Triển khai | Mặc định của OS | Static (hugetlbfs) hoặc THP | Bắt buộc cấu hình Static khi khởi động boot |
| Trường hợp sử dụng tối ưu | Ứng dụng văn phòng, Web nhẹ, script | Database thông dụng, KVM VM chuẩn | DPDK gói tin mạng, HFT, VM DB hàng trăm GB |

<!-- ORIGINAL 53 END -->
