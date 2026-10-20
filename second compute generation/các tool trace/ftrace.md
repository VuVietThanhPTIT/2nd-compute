- ftrace là tracer nội bộ của kernel là 1 framework gồm nhiều tiện ích khác nhau 
- khi tracefs được bật trong kernel thư mục /sys/kernel/tracing sẽ xuất hiện kết quả thông qua những thư mục đó và toàn bộ điều khiển đọc kết quả thong qua các thư mục đó 
- **Cơ chế** : ftrace là dynamic tracing . khi bắt đầu trace 1 hàm  , kernel code của hàm đó bị sửa đẻ thêm lệnh báo cho hệ thống tracing mỗi lần được gọi 

**Các file/tracer hay dùng:**
- `available_tracers` / `current_tracer`: xem và chọn tracer. ftrace chỉ chạy một tracer tại một thời điểm. [opensourceforu](https://www.opensourceforu.com/?p=11001)
- `function`: ghi lại lúc vào hàm. `function_graph`: ghi cả lúc vào và lúc thoát hàm, nên xem được call tree kèm thời gian. [missinglinkelectronics](https://lxr.missinglinkelectronics.com/linux+v2.6.38/Documentation/trace/ftrace.txt)
- `set_ftrace_filter`: giới hạn hàm cần trace. `set_ftrace_pid`: chỉ trace các thread có PID trong file. `set_graph_function`: chỉ trace hàm được liệt kê và các hàm nó gọi. [kernel](https://kernel.org/doc/Documentation/trace/ftrace.txt)
- `nop`: tracer rỗng, mặc định.


- Tài liệu gốc của kernel (Steven Rostedt): [https://kernel.org/doc/Documentation/trace/ftrace.txt](https://kernel.org/doc/Documentation/trace/ftrace.txt)
- Julia Evans, “getting started with ftrace” (dễ vào nhất, có ví dụ trace page fault): [https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/](https://jvns.ca/blog/2017/03/19/getting-started-with-ftrace/)
- Open Source For You, Kernel Tracing with ftrace Part 1: [https://www.opensourceforu.com/?p=11001](https://www.opensourceforu.com/?p=11001)
- Red Hat, dùng ftrace để trace latency: [https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/8/html/optimizing_rhel_8_for_real_time_for_low_latency_operation/assembly_using-the-ftrace-utility-to-trace-latencies_optimizing-rhel8-for-real-time-for-low-latency-operation](https://docs.redhat.com/en/documentation/red_hat_enterprise_linux_for_real_time/8/html/optimizing_rhel_8_for_real_time_for_low_latency_operation/assembly_using-the-ftrace-utility-to-trace-latencies_optimizing-rhel8-for-real-time-for-low-latency-operation)