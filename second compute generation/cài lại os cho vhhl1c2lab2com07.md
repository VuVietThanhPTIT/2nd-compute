![](Pasted%20image%2020261014172526.png)


```
bỏ raid chạy chay trên 1 disk, 1 disk để nguyên làm datastore nhé chú
os ubuntu 24.04, tạo phân vùng tầm 100GB để os thôi, phần còn lại để nguyên để làm block metadata nếu cần
```


![](Pasted%20image%2020261014172658.png)


```

Warning: Use the --scripts parameter to include the scripts.
warning: storcli-007.2207.0000.0000-1.noarch.rpm: Header V3 RSA/SHA256 Signature, key ID 26c2b797: NOKEY
        dpkg --no-force-overwrite -i storcli_007.2207.0000.0000-2_all.deb
Selecting previously unselected package storcli.
(Reading database ... 219787 files and directories currently installed.)
Preparing to unpack storcli_007.2207.0000.0000-2_all.deb ...
Unpacking storcli (007.2207.0000.0000-2) ...
Setting up storcli (007.2207.0000.0000-2) ...
root@vhhl1c2lab2com07:~# ^C
root@vhhl1c2lab2com07:~# S=$(find / -name storcli64 -type f 2>/dev/null | head -1); echo "S=$S"
S=/opt/MegaRAID/storcli/storcli64
root@vhhl1c2lab2com07:~# chmod +x "$S"
root@vhhl1c2lab2com07:~# $S -v

      StorCli SAS Customization Utility Ver 007.2207.0000.0000 Sep 22, 2022

    (c)Copyright 2022, Broadcom Inc. All Rights Reserved.



root@vhhl1c2lab2com07:~# $S /c0 show
Generating detailed summary of the adapter, it may take a while to complete.

CLI Version = 007.2207.0000.0000 Sep 22, 2022
Operating system = Linux 5.4.0-167-generic
Controller = 0
Status = Success
Description = None

Product Name = HPE MR216i-a Gen10+
Serial Number = PXBSA0B36HDMO4
PCI Slot Number = 12
SAS Address =  500062b211e5adc0
PCI Address = 00:47:00:00
System Time = 10/15/2026 01:55:48
Mfg. Date = 11/28/22
Controller Time = 10/15/2026 01:57:20
FW Package Build = 52.16.3-4455
BIOS Version = 7.16.00.0_0x07100501
FW Version = 5.160.03-3673
Driver Name = megaraid_sas
Driver Version = 07.713.01.00-rc1
Current Personality = JBOD-Mode
Vendor Id = 0x1000
Device Id = 0x10E2
SubVendor Id = 0x1590
SubDevice Id = 0x32C
Host Interface = PCI-E
Device Interface = SAS-12G
Bus Number = 71
Device Number = 0
Function Number = 0
Domain ID = 0
Security Protocol = None
Arrays = 1

TOPOLOGY :
========

------------------------------------------------------------------------------
DG Arr Row EID:Slot DID Type  State BT        Size PDC  PI SED DS3  FSpace TR
------------------------------------------------------------------------------
 0 -   -   -        -   RAID1 Optl  N  446.625 GiB dflt N  N   none N      N
 0 0   -   -        -   RAID1 Optl  N  446.625 GiB dflt N  N   none N      N
 0 0   0   252:1    0   DRIVE Onln  N    480.00 GB dflt N  N   none -      N
 0 0   1   252:2    1   DRIVE Onln  N    480.00 GB dflt N  N   none -      N
------------------------------------------------------------------------------

DG=Array|VD=Logical Drive|Slot=Drive Bay No|Span=Span Index|Row=Row Index|EID=Enclosure Device ID
DID=Device ID|Type=Physical Drive Type|Onln=Online|Rbld=Rebuild|Optl=Optimal|Dgrd=Degraded
Pdgd=Partially degraded|Offln=Offline|BT=Background Task Active
PDC=Drive Cache|PI=Data Integrity Info|SED=Self Encrypting Drive|Frgn=Foreign
DS3=Dimmer Switch 3|dflt=Default|Msng=Missing|FSpace=Free Space Present
TR=Transport Ready

Virtual Drives = 1

VD LIST :
=======

-----------------------------------------------------------------
DG/VD TYPE  State Access Consist Cache Cac sCC        Size Name
-----------------------------------------------------------------
0/239 RAID1 Optl  RW     Yes     NRWTD -   ON  446.625 GiB raid1
-----------------------------------------------------------------

DG=Arrays|VD=Virtual Drive/Logical Drive|Rec=Recovery
Cac=CacheCade|OfLn=OffLine|Pdgd=Partially Degraded|Dgrd=Degraded
Optl=Optimal|dflt=Default|RO=Read Only|RW=Read Write|HD=Hidden|TRANS=TransportReady
B=Blocked|Consist=Consistent|R=Read Ahead Always|NR=No Read Ahead|WB=WriteBack
AWB=Always WriteBack|WT=WriteThrough|C=Cached IO|D=Direct IO|sCC=Scheduled
Check Consistency

Physical Drives = 2

Drive LIST :
==========

--------------------------------------------------------------------------
EID:Slt DID State DG      Size Intf Med SED PI SeSz Model         Sp Type
--------------------------------------------------------------------------
252:1     0 Onln   0 480.00 GB SATA SSD N   N  512B VK000480GZXRF U  -
252:2     1 Onln   0 480.00 GB SATA SSD N   N  512B VK000480GZXRF U  -
--------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Drive Bay No|DID=Device ID|DG=Arrays
DHS=Dedicated Spare Drive|UGood=Unconfigured Good|GHS=Global Spare Drive
UBad=Unconfigured Bad|Sntze=Sanitize|Onln=Online|Offln=Offline|Intf=Interface
Med=Media Type|SED=Self Encryptive Drive|PI=Data Integrity Info
SeSz=Sector Size|Sp=Spun|U=Up|D=Down|T=Transition|F=Foreign
UGUnsp=UGood Unsupported|UGShld=UGood shielded|HSPShld=Spare shielded
CFShld=Configured shielded|Cpybck=CopyBack|CBShld=Copyback Shielded
UBUnsp=UBad Unsupported|Rbld=Rebuild

Enclosures = 1

Enclosure LIST :
==============

----------------------------------------------
EID State Slots PD PS Fans TSs Alms SIM Port#
----------------------------------------------
252 OK        8  2  0    0   0    0   0 1I
----------------------------------------------

Slot=Bay Count | SlotBP=Bays On Port | EID=Enclosure Device ID
PD=Drive Count | PS=Power Supply Count| SIM=SIM Count
TSs=Temperature Sensor Count |  Alms=Alarm Count | ProdID=Product ID



root@vhhl1c2lab2com07:~# $S /c0 show personality
CLI Version = 007.2207.0000.0000 Sep 22, 2022
Operating system = Linux 5.4.0-167-generic
Controller = 0
Status = Success
Description = None


Controller Properties :
=====================

-----------------------------------------
Ctrl_Prop                          Value
-----------------------------------------
Current Personality                JBOD
Behavior mode                      JBOD
Supported Auto Configure Behaviors JBOD
-----------------------------------------



root@vhhl1c2lab2com07:~# $S /c0/vall show
CLI Version = 007.2207.0000.0000 Sep 22, 2022
Operating system = Linux 5.4.0-167-generic
Controller = 0
Status = Success
Description = None


Virtual Drives :
==============

-----------------------------------------------------------------
DG/VD TYPE  State Access Consist Cache Cac sCC        Size Name
-----------------------------------------------------------------
0/239 RAID1 Optl  RW     Yes     NRWTD -   ON  446.625 GiB raid1
-----------------------------------------------------------------

DG=Arrays|VD=Virtual Drive/Logical Drive|Rec=Recovery
Cac=CacheCade|OfLn=OffLine|Pdgd=Partially Degraded|Dgrd=Degraded
Optl=Optimal|dflt=Default|RO=Read Only|RW=Read Write|HD=Hidden|TRANS=TransportReady
B=Blocked|Consist=Consistent|R=Read Ahead Always|NR=No Read Ahead|WB=WriteBack
AWB=Always WriteBack|WT=WriteThrough|C=Cached IO|D=Direct IO|sCC=Scheduled
Check Consistency



root@vhhl1c2lab2com07:~# $S /c0/eall/sall show
CLI Version = 007.2207.0000.0000 Sep 22, 2022
Operating system = Linux 5.4.0-167-generic
Controller = 0
Status = Success
Description = Show Drive Information Succeeded.


Drive Information :
=================

--------------------------------------------------------------------------
EID:Slt DID State DG      Size Intf Med SED PI SeSz Model         Sp Type
--------------------------------------------------------------------------
252:1     0 Onln   0 480.00 GB SATA SSD N   N  512B VK000480GZXRF U  -
252:2     1 Onln   0 480.00 GB SATA SSD N   N  512B VK000480GZXRF U  -
--------------------------------------------------------------------------

EID=Enclosure Device ID|Slt=Drive Bay No|DID=Device ID|DG=Arrays
DHS=Dedicated Spare Drive|UGood=Unconfigured Good|GHS=Global Spare Drive
UBad=Unconfigured Bad|Sntze=Sanitize|Onln=Online|Offln=Offline|Intf=Interface
Med=Media Type|SED=Self Encryptive Drive|PI=Data Integrity Info
SeSz=Sector Size|Sp=Spun|U=Up|D=Down|T=Transition|F=Foreign
UGUnsp=UGood Unsupported|UGShld=UGood shielded|HSPShld=Spare shielded
CFShld=Configured shielded|Cpybck=CopyBack|CBShld=Copyback Shielded
UBUnsp=UBad Unsupported|Rbld=Rebuild



root@vhhl1c2lab2com07:~# ^C
root@vhhl1c2lab2com07:~# $S /c0/e252/s1 show all | grep -Ei "SN =|WWN"
SN = S6MPNA0TB07403
WWN = 5002538F02B0712E
root@vhhl1c2lab2com07:~# $S /c0/e252/s2 show all | grep -Ei "SN =|WWN"
SN = S6MPNA0TB02705
WWN = 5002538F02B02634

```



```
B=/root/backup-$(hostname)-$(date +%F); mkdir -p $B

# 1. Config files (Netplan, DNS, NetworkManager, interfaces)
cp -a /etc/netplan $B/netplan 2>/dev/null
cp -a /etc/network/interfaces* $B/ 2>/dev/null
cp -a /etc/systemd/network $B/systemd-network 2>/dev/null
cp -a /etc/hosts /etc/resolv.conf $B/ 2>/dev/null
cp -a /etc/iproute2/rt_tables $B/ 2>/dev/null
resolvectl status > $B/resolvectl.txt 2>/dev/null

# 2. Interface IP, link, and policy routing
ip -br link            > $B/ip-link-brief.txt
ip -d link             > $B/ip-link-detail.txt
ip -br addr            > $B/ip-addr-brief.txt
ip addr                > $B/ip-addr-full.txt
ip rule show           > $B/ip-rule.txt
ip route show table all > $B/ip-route-all.txt
ip neigh show          > $B/ip-neighbors.txt

# 3. Hardware mapping, drivers, and permanent MACs
lspci -nnk -D | grep -i -A3 net > $B/lspci-network.txt
lshw -C network > $B/lshw-network.txt 2>/dev/null

for nic in $(ls /sys/class/net/); do
    echo "=== $nic ===" >> $B/ethtool-all.txt
    ethtool -i "$nic" 2>/dev/null >> $B/ethtool-drivers.txt
    ethtool -P "$nic" 2>/dev/null >> $B/ethtool-perm-mac.txt
done

# 4. Bonding & Bridges
cp -a /proc/net/bonding $B/bonding 2>/dev/null
bridge link > $B/bridge-link.txt 2>/dev/null
bridge fdb show > $B/bridge-fdb.txt 2>/dev/null

# 5. lldpd
lldpcli show neighbors details > $B/lldp-neighbors.txt 2>/dev/null
lldpcli show interfaces detail > $B/lldp-interfaces.txt 2>/dev/null
lldpcli show configuration     > $B/lldp-config.txt 2>/dev/null
lldpctl -f json                > $B/lldp-neighbors.json 2>/dev/null
cp -a /etc/lldpd.conf /etc/lldpd.d /etc/default/lldpd $B/ 2>/dev/null

# 6. Firewall & Sysctl
iptables-save > $B/iptables.rules 2>/dev/null
ip6tables-save > $B/ip6tables.rules 2>/dev/null
nft list ruleset > $B/nftables.rules 2>/dev/null
sysctl -a 2>/dev/null | grep "^net\." > $B/sysctl-net.txt

# 7. Package and inspect
tar czf $B.tar.gz -C /root $(basename $B)
ls -lh $B.tar.gz

echo "Done. Remember to copy $B.tar.gz off the server before reinstalling!"
```


```
cp /root/backup-*.tar.gz /tmp/ chmod 644 /tmp/backup-*.tar.gz
scp fciadm@vhhl1c2lab2com07:/tmp/backup-*.tar.gz ~/

```


---- 
## 1 , Tải file iso 24.04 live server
- mount qua ilo 

- bấm reset qua ilo menu 
![](Pasted%20image%2020261015103706.png)


![](Pasted%20image%2020261015104124.png)


![](Pasted%20image%2020261015104707.png)


![](Pasted%20image%2020261015104756.png)

![](Pasted%20image%2020261015104914.png)


![](Pasted%20image%2020261015104939.png)


![](Pasted%20image%2020261015105031.png)


![](Pasted%20image%2020261015105422.png)


- Sau đó exit và bấm try 
![](Pasted%20image%2020261015105623.png)

![](Pasted%20image%2020261015110458.png)

custom 

![](Pasted%20image%2020261015110925.png)
\

![](Pasted%20image%2020261015111451.png)


--- 
- Copy file cấu hình mạng từ máy Window lên ilo qua virtual folder 
- ![](Pasted%20image%2020261015134343.png)