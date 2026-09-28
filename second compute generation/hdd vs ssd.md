- hdd : 1 đĩa từ gồm nhiều vòng , mỗi vòng tròn trên đĩa gọi là1 track , đầu đọc ghi có thể di chuyển tới bất kỳ phần vòng nào 
- mỗi phần của 1 track (  1 track được chia theo các góc gọi là sector ) - **sector** là đơn vị nhỏ nhất của đĩa ( có thể chứa 512 - 4096 byte)
  -> Thời gian đọc đầu đọc tốn thời gian : cần thời gian tìm đùng track và cần tìm đúng sector 


sdd 
- ![[Pasted image 20260927090710.png]]![[Pasted image 20260927091035.png]]
- 1 Channel — đường truyền vật lý riêng biệt đến controller của ssd 
- 1 chip ( bên trong channel )
- 1 die ( nhiều bên trong 1 chip )
- 1 plane ( 1- 6 plane bên trong 1 die  )
- 1 block (  1 trăm tới hàng nghìn block  trên 1 plane  ) đơn vị nhỏ nhất để xóa 
- 1 page ( 1 - hàng trăm page trên  1 block , 8 - 16 KB / page , là cái để ghi  )
- 1 cell ( là cái giữ điện tích )