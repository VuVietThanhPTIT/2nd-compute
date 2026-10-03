
![[Pasted image 20261003154507.png]]
# 1 , Kiến trúc 3 phần : 
- Driver ( front - end ) : nằm trong kernel của guest , nhận I/ O từ tiến trình người dùng , chuyển cho device , rồi nhận kq 
- Device ( backend ) nằm trong hypervisor ( ở đây là qemu ) nhận request và giao việc cho phần cứng của host 
- virtualqueue : queue hàng đợi yêu cầu được cả driver và device cùng nhìn thấy trong ram   , vring là cách implent code hàng đợi đó 
Figure 3 below shows Qemu’s version of the VirtQueue and VRing data structures.
![[Pasted image 20261003155054.png]]

For example, figure 4 below shows the Linux kernel’s version of its VirtQueue and VRing data structures.![[Pasted image 20261003161134.png]]
- VRings :
	- Each VirtQueue can have up to, and usually does, three types of VRings (or areas):
		- Descriptor ring (descriptor area)
		- Available ring (driver area)
		- Used ring (device area)


### **The VirtIO Driver Operations**
- virio blk or scsi convert the block i/o request into a desscriptor chain and submits it to a virtual queue 
- ![[Pasted image 20261003225545.png]]
The **Descriptor Table** contains descriptors (virtq_desc) that describe guest memory buffers through fields such as the guest-physical address (addr), buffer length (len), descriptor flags (flags), and an optional pointer to the next descriptor in the chain (next).

By linking descriptors in the next field, the driver can construct descriptor chains that allow a single I/O request to reference multiple buffers, such as a request header, a data buffer, and a completion status buffer.

The **Available Ring** is written by the driver and read by the device. Each ring entry stores the index of the head descriptor of a submitted descriptor chain. After placing the head descriptor index into the available ring and updating the available ring index (idx), the driver notifies that a new request is available for processing.

The **Used Ring** is written by the device (QEMU) and read by the driver. It's used to notify the driver that the kernel storage stack has completed the I/O request.

Using shared guest-memory mappings, it's possible to access the guest buffers directly while minimizing unnecessary data copies and latency.





[Understanding Disk I/O in QEMU/KVM with VirtIO | Veeam Community Resource Hub](https://community.veeam.com/blogs-and-podcasts-57/understanding-disk-i-o-in-qemu-kvm-with-virtio-13460)