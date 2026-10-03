[ ỨNG DỤNG USER SPACE ]
         │ (System Calls: open, read, write, statx)
         ▼
[ VIRTUAL FILE SYSTEM (VFS) & PAGE CACHE ]
         │
         ├── File Systems: ext4, XFS, Btrfs, NFS, /proc, tmpfs, FUSE...
         └── Storage Targets: Linux I/O Target (LIO - iSCSI), NVMe Target
         │
         ▼ (Chuyển đổi File I/O -> Block I/O)
[ TẦNG BIO (struct bio) ]
         │
         ├── STACKED BLOCK DEVICES / DEVICE MAPPER (Không áp dụng I/O Scheduler)
         │     ├── Software RAID (md)
         │     ├── DRBD (Network Block Replication)
         │     ├── LVM (Logical Volume Manager)
         │     └── DM-Crypt (Mã hóa LUKS)
         │
         ▼
[ BLOCK LAYER: blk-mq (Multi-Queue Architecture) ]
         │
         ├── Software Queues (Theo từng CPU Core): Merge, Reorder, I/O Scheduler (none, mq-deadline, bfq, kyber)
         └── Hardware Queues (Khóa độc lập, map xuống bộ điều khiển)
         │
         ▼ (struct request)
[ DEVICE DRIVERS ]
         ├── NVMe Driver (Giao tiếp thẳng PCIe)
         ├── SCSI Subsystem (sd_mod, SCSI Core, SATA/SAS HBA)
         └── MTD Layer (Raw Flash / Embedded)
         │
         ▼
[ PHẦN CỨNG LƯU TRỮ ] (NVMe SSD, SATA SSD, HDD cơ, Chip NAND Flash)