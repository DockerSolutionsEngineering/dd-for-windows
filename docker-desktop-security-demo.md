
# Docker Desktop Security Demo

This document demonstrates the security boundary preventing privileged Linux containers from gaining privileged access to the Mac/Windows/Linux host running Docker Desktop. Note this document only applies to Linux containers, not Windows native containers (e.g. IIS).


# Filesystem security

Docker containers can share host filesystems with commands like


```
PS C:\Users\dave> docker run -it -v C:\Users\dave\myproject:/myproject alpine
/ # ls /myproject
```


The Docker containers run inside a Linux VM which by default has no access to the host. There needs to be a fileserver on the host to provide access to host files. Docker Desktop uses a different technology depending on the OS configuration, however **in all cases the host fileserver applies access controls based on the Host user account which runs Docker Desktop, not the Linux userid/groupid inside the container.**

For example if root in a privileged container tries to write to a Host system directory it will be rejected by the fileserver on Windows:


```
PS C:\Users\dave> docker run -it --privileged -v C:\Windows:/windows alpine
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)
/ # touch /windows/hello.txt
touch: /windows/hello.txt: Permission denied
```


Equivalently on Mac, imagine there is a directory which only root can read or write:


```
djs55@m1 / % ls -l /tmp/
total 32
…
drwx------  3 root   wheel    96 28 Jul 13:31 test
…
djs55@m1 / % ls -l /tmp/test
total 0
ls: /tmp/test: Permission denied
djs55@m1 / % sudo ls -l /tmp/test
Password:
total 8
-rw-r--r--  1 root  wheel  6 28 Jul 13:32 hello
djs55@m1 / % sudo cat /tmp/test/hello
hello
```


Attempts to read or write it from containers will fail, even if the user is root inside the container:


```
djs55@m1 / % docker run -it --privileged -v /tmp:/host_tmp alpine sh
/ # id
uid=0(root) gid=0(root) groups=0(root),1(bin),2(daemon),3(sys),4(adm),6(disk),10(wheel),11(floppy),20(dialout),26(tape),27(video)

/ # ls -l /host_tmp
total 16
…
drwx------    3 root     root            96 Jul 28 12:31 test
…

/ # ls -l /host_tmp/test
total 0

/ # cat /host_tmp/test/hello
cat: can't open '/host_tmp/test/hello': Operation not permitted
```



# Network security

Linux containers run inside a Virtual Machine which has an entirely different network stack to the host. For example my host has:


```
PS C:\Users\dave> ipconfig

Windows IP Configuration

Ethernet adapter vEthernet (Internal):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::5d2b:d14:8d7e:48a4%21
   Autoconfiguration IPv4 Address. . : 169.254.72.164
   Subnet Mask . . . . . . . . . . . : 255.255.0.0
   Default Gateway . . . . . . . . . :

Ethernet adapter Ethernet:

   Connection-specific DNS Suffix  . : localdomain
   Link-local IPv6 Address . . . . . : fe80::a449:3d9b:5b1d:1fda%8
   IPv4 Address. . . . . . . . . . . : 192.168.1.148
   Subnet Mask . . . . . . . . . . . : 255.255.255.0
   Default Gateway . . . . . . . . . : 192.168.1.1

Wireless LAN adapter WiFi:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :

Wireless LAN adapter Local Area Connection* 9:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :

Wireless LAN adapter Local Area Connection* 10:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :

Ethernet adapter Bluetooth Network Connection:

   Media State . . . . . . . . . . . : Media disconnected
   Connection-specific DNS Suffix  . :

Ethernet adapter vEthernet (vEthernet (Inte):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::9d59:8b00:a168:f3e1%12
   IPv4 Address. . . . . . . . . . . : 172.21.128.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter vEthernet (Default Switch):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::3d38:2413:7380:ebf1%31
   IPv4 Address. . . . . . . . . . . : 172.24.0.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter vEthernet (Ethernet):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::8552:5726:4849:9b67%37
   IPv4 Address. . . . . . . . . . . : 172.20.32.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter vEthernet (WiFi):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::c448:d3e5:44ac:f808%42
   IPv4 Address. . . . . . . . . . . : 172.20.48.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter vEthernet (WSL):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::7c89:7245:abec:ddeb%87
   IPv4 Address. . . . . . . . . . . : 172.26.96.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :

Ethernet adapter vEthernet (nat):

   Connection-specific DNS Suffix  . :
   Link-local IPv6 Address . . . . . : fe80::3ccb:df0b:4754:812a%92
   IPv4 Address. . . . . . . . . . . : 172.24.208.1
   Subnet Mask . . . . . . . . . . . : 255.255.240.0
   Default Gateway . . . . . . . . . :
```


Meanwhile a privileged Linux container in Linux’s “host” network namespace (where “host” really means VM in this case) can see:


```
PS C:\Users\dave> docker run -it --net=host --pid=host alpine
/ # ifconfig
cni0      Link encap:Ethernet  HWaddr D2:23:B4:59:10:AA
          inet addr:10.1.0.1  Bcast:10.1.255.255  Mask:255.255.0.0
          inet6 addr: fe80::d023:b4ff:fe59:10aa/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:9688 errors:0 dropped:0 overruns:0 frame:0
          TX packets:10202 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:1367152 (1.3 MiB)  TX bytes:2366722 (2.2 MiB)

docker0   Link encap:Ethernet  HWaddr 02:42:A6:A8:B5:3C
          inet addr:172.17.0.1  Bcast:172.17.255.255  Mask:255.255.0.0
          inet6 addr: fe80::42:a6ff:fea8:b53c/64 Scope:Link
          UP BROADCAST MULTICAST  MTU:1500  Metric:1
          RX packets:20069 errors:0 dropped:0 overruns:0 frame:0
          TX packets:53742 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:812313 (793.2 KiB)  TX bytes:76938256 (73.3 MiB)

eth0      Link encap:Ethernet  HWaddr 02:50:00:00:00:01
          inet addr:192.168.65.3  Bcast:192.168.65.255  Mask:255.255.255.0
          inet6 addr: fe80::50:ff:fe00:1/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:53734 errors:0 dropped:0 overruns:0 frame:0
          TX packets:20100 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:76937428 (73.3 MiB)  TX bytes:1095633 (1.0 MiB)

lo        Link encap:Local Loopback
          inet addr:127.0.0.1  Mask:255.0.0.0
          inet6 addr: ::1/128 Scope:Host
          UP LOOPBACK RUNNING  MTU:65536  Metric:1
          RX packets:288569 errors:0 dropped:0 overruns:0 frame:0
          TX packets:288569 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:1000
          RX bytes:59354419 (56.6 MiB)  TX bytes:59354419 (56.6 MiB)

services1 Link encap:Ethernet  HWaddr 46:B8:1D:E7:37:B6
          inet addr:192.168.65.4  Bcast:0.0.0.0  Mask:255.255.255.255
          inet6 addr: fe80::44b8:1dff:fee7:37b6/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:311 errors:0 dropped:0 overruns:0 frame:0
          TX packets:313 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:24817 (24.2 KiB)  TX bytes:21428 (20.9 KiB)

veth01bf2896 Link encap:Ethernet  HWaddr 46:C4:53:84:FE:BE
          inet6 addr: fe80::44c4:53ff:fe84:febe/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1974 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1941 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:189034 (184.6 KiB)  TX bytes:192130 (187.6 KiB)

veth11c3b3cf Link encap:Ethernet  HWaddr E2:8C:6B:25:31:CE
          inet6 addr: fe80::e08c:6bff:fe25:31ce/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:5559 errors:0 dropped:0 overruns:0 frame:0
          TX packets:6154 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:1109808 (1.0 MiB)  TX bytes:1958641 (1.8 MiB)

veth6af0a0f9 Link encap:Ethernet  HWaddr 6A:6B:C2:73:41:12
          inet6 addr: fe80::686b:c2ff:fe73:4112/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:196 errors:0 dropped:0 overruns:0 frame:0
          TX packets:206 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:16177 (15.7 KiB)  TX bytes:27241 (26.6 KiB)

veth7dfeff33 Link encap:Ethernet  HWaddr 4A:EA:4B:88:F7:C4
          inet6 addr: fe80::48ea:4bff:fe88:f7c4/64 Scope:Link
          UP BROADCAST RUNNING MULTICAST  MTU:1500  Metric:1
          RX packets:1961 errors:0 dropped:0 overruns:0 frame:0
          TX packets:1956 errors:0 dropped:0 overruns:0 carrier:0
          collisions:0 txqueuelen:0
          RX bytes:187849 (183.4 KiB)  TX bytes:192864 (188.3 KiB)

/ #
```


The host networks and interfaces are all independent from the VM networks and interfaces. The network stack seen by Linux containers is entirely virtual. All outgoing traffic from the VM is sent over a secure shared-memory tunnel to the host `com.docker.vpnkit.exe` process, where host firewall rules are applied before the traffic is transmitted via socket `connect()`, `read()`, `write()`, `close()` calls. Since the host network interfaces are not visible inside the VM, it is not possible even for a privileged Linux process to interact with them. See the separate Docker Desktop Networking document for more details.


# Memory isolation

Linux containers are always run inside a Linux VM.

On Windows in Hyper-V mode the helper “DockerDesktopVM” can be seen in Hyper-V manager. On Windows in WSL 2 mode there is a hidden helper VM managed by the WSL 2 subsystem. In both cases the Hyper-V hypervisor (as used in Azure) is used to guarantee memory isolation between the host and the VM.

On Mac the VM is managed by macOS via the hypervisor.framework and virtualization.framework. The macOS hypervisor is used to guarantee memory isolation between the host and the VM.

On Linux the VM is managed by KVM. The Linux kernel is used to guarantee memory isolation between the host and the VM.
