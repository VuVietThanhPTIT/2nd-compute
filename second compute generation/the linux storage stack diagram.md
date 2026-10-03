
![[Linux-storage-stack-diagram_v6.18_page_1.png]]

https://youtu.be/hCKruOlLPIQ?si=pd88Eyu8xZwXDcuC

![[Pasted image 20261001140616.png]]
![[Pasted image 20261001140733.png]]
![[Pasted image 20261001140747.png]]

![[Pasted image 20261001140939.png]]
![[Pasted image 20261001140944.png]]


Luồng đọc : 
App : read ( fd , buf , n ) với vị trí 5000 <- API posix qua libc 
( byte 5000 nằm ở trang 1 của file : )
- libc -> syscal read -> kernel 
- fd -> struct file ( tạo ra từ lúc open ) -> f_op -> read iter -> hàm đọc 
