![Pasted image 20261001084618](img/Pasted%20image%2020261001084618.png)

The Linux kernel operates as the bridge between system hardware and user-space applications. At its core, it is organized into three major functional pillars—**Process Management**, **Memory Management**, and the **I/O Subsystem**—all accessed via the **System Call Interface (SCI)**.
### Core Subsystems of the Linux Kernel

#### 1. System Call Interface (SCI)

The top layer of the kernel space serving as the front door for user programs. When user-space code (often via `glibc`) requests system services like reading a file (`read`) or spawning a process (`fork`), the request transitions through the SCI from unprivileged User Mode to privileged Kernel Mode.
#### 2. Process Management Subsystem

Responsible for managing CPU execution resources and process lifecycles:
- **Process Scheduler:** Determines which process or thread runs on available CPU cores and for how long (traditionally using the Completely Fair Scheduler / CFS).
- **Process/Thread Creation & Termination:** Manages task structures (`task_struct`), process cloning, execution states, and resource cleanup upon exit.
- **Signal Handling:** Delivers asynchronous event notifications (such as `SIGTERM`, `SIGKILL`, or `SIGSEGV`) to processes.
#### 3. Memory Management Subsystem

Ensures programs have safe, isolated, and efficient access to system RAM:
- **Virtual Memory:** Allocates each process its own continuous virtual address space, mapped to physical memory pages via hardware page tables (MMU).
- **Paging & Page Replacement:** Automatically swaps inactive memory pages between RAM and swap storage when memory pressure occurs.
- **Page Cache:** Keeps recently accessed disk files in unused RAM to accelerate future reads and writes.
#### 4. I/O Subsystem

Handles communication with storage devices, peripheral hardware, and networking interfaces:
- **Virtual File System (VFS):** Provides a standard POSIX abstraction layer (files, directories, inodes) so user applications interact identically with ext4, XFS, NFS, or special pseudo-filesystems like `/proc` and `/sys`.
- **Block Storage Pipeline:** Flows from the **File systems** through the **Generic block layer** and the **I/O Scheduler** down to **Block device drivers** (e.g., NVMe, SATA).
- **Network Stack:** Routes data through **Sockets**, filtering via **Netfilter / Nftables**, through core **Network protocols** (TCP, UDP, IP), managed by the **Packet Scheduler**, and sent out through **Network device drivers**.

- **Character Devices & Terminals:** Manages unbuffered streams through **Line discipline** and **Character device drivers** (keyboards, serial ports, `/dev/null`).
#### 5. IRQs & Dispatcher

At the lowest layer interfacing with the hardware:
- **IRQs (Interrupt Requests):** Hardware devices trigger electrical interrupts to notify the kernel of events (such as packet arrival or keystrokes). The kernel processes these via Interrupt Service Routines (ISRs) and deferrable bottom-halves (SoftIRQs/Tasklets).
- **Dispatcher:** Performs low-level context switches to load registers and memory page tables for the next scheduled process.