# Phần 1 — Giải thích các thông số benchmark trước khi vào kịch bản

## IOPS

Số lệnh I/O hoàn thành mỗi giây. Với block size 4K, đây gần như tương đương với "hệ thống chịu được bao nhiêu request nhỏ mỗi giây" — đúng loại workload database, hay nhiều VM cùng ghi log.

## Percentile latency (P50/P90/P99/P99.99)

Đây là cách trả lời câu hỏi "trong N request, request tệ thứ mấy mất bao lâu":

| Percentile   | Ý nghĩa                                                                           |
| ------------ | --------------------------------------------------------------------------------- |
| P50 (median) | Một nửa số request nhanh hơn giá trị này. Đại diện cho trải nghiệm "bình thường". |
| P90          | 90% request nhanh hơn giá trị này, 10% chậm hơn.                                  |
| P99          | Trong 100 request, request tệ thứ 99 mất bao lâu. Chỉ 1% tệ hơn mức này.          |
| P99.99       | Trong 10.000 request, chỉ 1 request tệ hơn mức này.                               |

**Vì sao tập trung vào P99/P99.99 thay vì trung bình:** trung bình bị "lu mờ" bởi số đông. Ví dụ: 999 request mất 0.3ms, 1 request mất 50ms (do garbage collection SSD, do retransmit TCP, do tranh chấp CPU) — trung bình vẫn đẹp, nhưng P99.99 sẽ lộ ra ngay. Với hệ thống chạy hàng triệu request/ngày, "trường hợp hiếm" này xảy ra thường xuyên về số tuyệt đối. Đây đúng là thứ đề bài gốc của team ("tail latency thua metal vài chục %") đang nhắm tới.

## Queue depth (iodepth) và vì sao phải quét theo từng mức

Số lượng I/O "đang bay" (đã gửi, chưa nhận kết quả) cùng lúc. Quét qua nhiều mức (1, 2, 4, 8, 16, 32, 64...) cho bạn một đường cong, không phải một điểm:

- **qd thấp:** đo được latency thật của một request đơn lẻ (gần với "tốc độ ánh sáng" của hệ thống).
- **qd tăng dần:** IOPS tăng theo (tận dụng độ trễ ẩn — trong lúc chờ request A về thì gửi thêm request B, C...) cho tới khi phần cứng/network bão hoà.
- **qd vượt điểm bão hoà:** IOPS gần như đi ngang, nhưng latency tăng vọt (vì request phải xếp hàng chờ) — đây chính là điều bạn đã thấy: qd128 chỉ hơn qd1 khoảng 12 lần IOPS dù song song gấp 128 lần, vì hệ thống bão hoà từ khoảng qd12.

## Block size 4K

Đây là block size chuẩn để đo IOPS (khác với đo throughput dùng block lớn 1M). Lý do: với block nhỏ, chi phí cố định trên mỗi request (network round-trip, xử lý CPU, ngắt) chiếm phần lớn thời gian — số liệu phản ánh đúng "trần số lượng request xử lý được", không bị chi phí truyền dữ liệu che lấp.

## direct=1

Bắt buộc để bỏ qua page cache (dùng cờ O_DIRECT) — không có cờ này, lần đọc lặp lại có thể chỉ trả lời từ RAM thay vì chạm thật xuống đĩa/network.

## sync=0 — cần làm rõ một điểm dễ nhầm

Đây là tham số riêng của fio, khác với tên ioengine "sync". `sync=0` (mặc định) nghĩa là **không ép ghi qua O_SYNC** sau mỗi lần write — nó chỉ ảnh hưởng tới **ghi qua buffered I/O**. Với bài test bạn đang làm là **randread + direct=1**, cờ `sync` gần như không có tác dụng gì (vì đã dùng O_DIRECT, không còn buffered I/O để mà ép sync). Mình vẫn khai báo tường minh `sync=0` để không phụ thuộc giá trị mặc định, phòng khi sau này bạn đổi sang test ghi.

**Về ioengine:** để quét được nhiều mức queue depth trong cùng 1 kịch bản có thể so sánh, dùng cố định `ioengine=libaio` cho mọi điểm (kể cả qd=1) — không cần đổi engine riêng cho từng điểm.

---
