---
title: nmap网络扫描工具基本用法
p: linux/nmap_usage.md
date: 2020-07-07 11:35:50
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# nmap网络扫描工具基本用法

nmap一般就用来扫描主机是否在线(特别是扫描局域网内存活的机器)、开放了哪些端口。其他的功能用的比较少，做渗透的人可能要了解的多些。

## 选项说明

nmap需要自行安装。

```
shell> yum -y install nmap
```

使用`nmap -h`可以查看选项和用法。选项非常多，这是功能强大的工具带来的必然结果，但简单使用并用不到几个选项。

```
Usage: nmap [Scan Type(s)] [Options] {target specification}

TARGET SPECIFICATION:
  Can pass hostnames, IP addresses, networks, etc.
  Ex: scanme.nmap.org, microsoft.com/24, 192.168.0.1; 10.0.0-255.1-254
  ★★★★
  -iL <inputfilename>: Input from list of hosts/networks
  -iR <num hosts>: Choose random targets
  --exclude <host1[,host2][,host3],...>: Exclude hosts/networks
  --excludefile <exclude_file>: Exclude list from file

HOST DISCOVERY:
  -sL: List Scan - simply list targets to scan
  ★★★★
  -sn: Ping Scan - disable port scan
  -Pn: Treat all hosts as online -- skip host discovery
  -PS/PA/PU/PY[portlist]: TCP SYN/ACK, UDP or SCTP discovery to given ports
  ★★★★
  -PE/PP/PM: ICMP echo, timestamp, and netmask request discovery probes
  -PO[protocol list]: IP Protocol Ping
  -PR: ARP ping - does not need HW address -> IP translation
  ★★★★
  -n/-R: Never do DNS resolution/Always resolve [default: sometimes]
  --dns-servers <serv1[,serv2],...>: Specify custom DNS servers
  --system-dns: Use OS's DNS resolver
  --traceroute: Trace hop path to each host

SCAN TECHNIQUES:
  ★★★★
  -sS/sT/sA/sW/sM: TCP SYN/Connect()/ACK/Window/Maimon scans  
  -sU: UDP Scan
  -sN/sF/sX: TCP Null, FIN, and Xmas scans
  --scanflags <flags>: Customize TCP scan flags
  -sI <zombie host[:probeport]>: Idle scan
  -sY/sZ: SCTP INIT/COOKIE-ECHO scans
  ★★★★
  -sO: IP protocol scan
  -b <FTP relay host>: FTP bounce scan

PORT SPECIFICATION AND SCAN ORDER:
  ★★★★
  -p <port ranges>: Only scan specified ports
        Ex: -p22; -p1-65535; -p U:53,111,137,T:21-25,80,139,8080,S:9
  -F: Fast mode - Scan fewer ports than the default scan
  -r: Scan ports consecutively - don't randomize
  --top-ports <number>: Scan <number> most common ports
  --port-ratio <ratio>: Scan ports more common than <ratio>

SERVICE/VERSION DETECTION:
  -sV: Probe open ports to determine service/version info
  -sR: Check what service uses opened ports using RPC scan
  --version-intensity <level>: Set from 0 (light) to 9 (try all probes)
  --version-light: Limit to most likely probes (intensity 2)
  --version-all: Try every single probe (intensity 9)
  --version-trace: Show detailed version scan activity (for debugging)

SCRIPT SCAN:
  -sC: equivalent to --script=default
  --script=<Lua scripts>: <Lua scripts> is a comma separated list of directories, script-files or script-categories
  --script-args=<n1=v1,[n2=v2,...]>: provide arguments to scripts
  --script-trace: Show all data sent and received
  --script-updatedb: Update the script database.

OS DETECTION:
  -O: Enable OS detection
  --osscan-limit: Limit OS detection to promising targets
  --osscan-guess: Guess OS more aggressively

TIMING AND PERFORMANCE:
  Options which take <time> are in seconds, or append 'ms' (milliseconds),
  's' (seconds), 'm' (minutes), or 'h' (hours) to the value (e.g. 30m).
  
  -T<0-5>: Set timing template (higher is faster)
  ★★★★
  --min-hostgroup/max-hostgroup <size>: Parallel host scan group sizes
  ★★★★
  --min-parallelism/max-parallelism <numprobes>: Probe parallelization
  --min-rtt-timeout/max-rtt-timeout/initial-rtt-timeout <time>: Specifies
      probe round trip time.
  --max-retries <tries>: Caps number of port scan probe retransmissions.
  --host-timeout <time>: Give up on target after this long
  --scan-delay/--max-scan-delay <time>: Adjust delay between probes
  --min-rate <number>: Send packets no slower than <number> per second
  --max-rate <number>: Send packets no faster than <number> per second

FIREWALL/IDS EVASION AND SPOOFING:
  -f; --mtu <val>: fragment packets (optionally w/given MTU)
  -D <decoy1,decoy2[,ME],...>: Cloak a scan with decoys
  ★★★★
  -S <IP_Address>: Spoof source address
  -e <iface>: Use specified interface
  -g/--source-port <portnum>: Use given port number
  --data-length <num>: Append random data to sent packets
  --ip-options <options>: Send packets with specified ip options
  --ttl <val>: Set IP time-to-live field
  --spoof-mac <mac address/prefix/vendor name>: Spoof your MAC address
  --badsum: Send packets with a bogus TCP/UDP/SCTP checksum

OUTPUT:
  ★★★★
  -oN/-oX/-oS/-oG <file>: Output scan in normal, XML, s|<rIpt kIddi3,
       and Grepable format, respectively, to the given filename.
  -oA <basename>: Output in the three major formats at once
  ★★★★
  -v: Increase verbosity level (use -vv or more for greater effect)
  -d: Increase debugging level (use -dd or more for greater effect)
  --reason: Display the reason a port is in a particular state
  --open: Only show open (or possibly open) ports
  --packet-trace: Show all packets sent and received
  --iflist: Print host interfaces and routes (for debugging)
  --log-errors: Log errors/warnings to the normal-format output file
  --append-output: Append to rather than clobber specified output files
  --resume <filename>: Resume an aborted scan
  --stylesheet <path/URL>: XSL stylesheet to transform XML output to HTML
  --webxml: Reference stylesheet from Nmap.Org for more portable XML
  --no-stylesheet: Prevent associating of XSL stylesheet w/XML output

MISC:
  -6: Enable IPv6 scanning
  -A: Enable OS detection, version detection, script scanning, and traceroute
  --datadir <dirname>: Specify custom Nmap data file location
  --send-eth/--send-ip: Send using raw ethernet frames or IP packets
  --privileged: Assume that the user is fully privileged
  --unprivileged: Assume the user lacks raw socket privileges
  -V: Print version number
  -h: Print this help summary page.

EXAMPLES:
  nmap -v -A scanme.nmap.org
  nmap -v -sn 192.168.0.0/16 10.0.0.0/8
  nmap -v -iR 10000 -Pn -p 80
```

常用的就上面标`★★★★`的几个。下面是解释：

```
-iL <inputfilename>:从输入文件中读取主机或者IP列表作为探测目标
-sn: PING扫描，但是禁止端口扫描。默认总是会扫描端口。禁用端口扫描可以加速扫描主机
-n/-R: 永远不要/总是进行DNS解析，默认情况下有时会解析
-PE/PP/PM:分别是基于echo/timestamp/netmask的ICMP探测报文方式。使用echo最快
-sS/sT/sA/sW:TCP SYN/Connect()/ACK/Window，其中sT扫描表示TCP扫描
-sU:UDP扫描
-sO:IP扫描
-p <port ranges>: 指定扫描端口
--min-hostgroup/max-hostgroup <size>: 对目标主机进行分组然后组之间并行扫描
--min-parallelism/max-parallelism <numprobes>: 设置并行扫描的探针数量
-oN/-oX/ <file>: 输出扫描结果到普通文件或XML文件中。输入到XML文件中的结果是格式化的结果
-v:显示详细信息，使用-vv或者更多的v显示更详细的信息
```

## 尝试一次扫描

nmap扫描一般会比较慢，特别是扫描非本机的时候。

```
[root@server2 ~]# nmap 127.0.0.1

Starting Nmap 6.40 ( http://nmap.org ) at 2017-06-20 13:03 CST
Nmap scan report for localhost (127.0.0.1)
Host is up (0.0000010s latency).
Not shown: 998 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
25/tcp open  smtp
```

只扫描出了两个端口，但是不代表真的只开了两个端口，这样不加任何参数的nmap将自动决定扫描1000个高危端口，但哪些是高危端口由nmap决定。从结果中也能看出来，`NOT shown:998 closed ports`表示998个关闭的端口未显示出来，随后又显示了2个open端口，正好1000个。虽说默认只扫描1000个，但常见的端口都能扫描出来。

从虚拟机扫描win主机看看。可以感受到，扫描速度明显降低了。

```
[root@server2 ~]# nmap 192.168.0.122
 
Starting Nmap 6.40 ( http://nmap.org ) at 2017-06-20 13:11 CST
Nmap scan report for 192.168.0.122
Host is up (1.2s latency).
Not shown: 990 closed ports
PORT     STATE    SERVICE
21/tcp   open     ftp
135/tcp  open     msrpc
139/tcp  open     netbios-ssn
443/tcp  open     https
445/tcp  open     microsoft-ds
514/tcp  filtered shell
902/tcp  open     iss-realsecure
912/tcp  open     apex-mesh
1583/tcp open     simbaexpress
5357/tcp open     wsdapi
 
Nmap done: 1 IP address (1 host up) scanned in 8.38 seconds
```

可以指定`-p [1-65535]`来扫描所有端口，或者使用`-p-`选项也是全面扫描。

```
[root@xuexi ~]# nmap -p- 127.0.0.1
```

nmap默认总是会扫描端口，可以使用-sn选项禁止扫描端口，以加速扫描主机是否存活。

## 扫描目标说明

Nmap支持CIDR风格的地址，Nmap将会扫描所有和**该参考IP地址**具有相同cidr位数的所有IP地址或主机。

例如192.168.10.0/24将扫描192.168.10.0和192.168.10.255之间的256台主机，192.168.10.40/24会做同样的事情。假设主机scanme.nmap.org的IP地址是205.217.153.62，scanme.nmap.org/16将扫描205.217.0.0和205.217.255.255之间的65536个IP地址。掩码位所允许的最小值是/1，这将会扫描半个互联网，最大值是/32，这将会扫描该主机或IP地址，因为所有主机位都固定了。

CIDR标志位很简洁但有时候不够灵活。例如也许想要扫描192.168.0.0/16，但略过任何以".0"或者".255"结束的IP地址，因为它们通常是网段地址或广播地址。可以用逗号分开的数字或范围列表为IP地址指定它的范围。例如"192.168.0-255.1-254"将略过该范围内以".0"和".255"结束的地址。范围不必限于最后的8位："0-255.0-255.13.37"将在整个互联网范围内扫描所有以"13.37"结束的地址。

Nmap命令行接受多个主机说明，它们不必是相同类型。如：

```
nmap www.hostname.com 192.168.0.0/8 10.0.0,1,3-7.0-255
```

虽然目标通常在命令行指定，下列选项也可用来控制目标的选择：

- `-iL <inputfilename>` (从列表中输入)

从`<inputfilename>`中读取目标说明。在命令行输入一堆主机名显得很笨拙，然而经常需要这样。例如DHCP服务器可能导出10000个当前租约列表。列表中的项可以是Nmap在命令行上接受的任何格式(IP地址，主机名，CIDR，IPv6，或者八位字节范围)。每一项必须以一个或多个空格、制表符或换行符分开。如果希望Nmap从标准输入读取列表，则使用`-`作为表示/dev/stdin。

- `--exclude <host1[，host2][，host3]，...>` (排除主机/网络)
- `--excludefile <excludefile>` (排除文件中的列表)，这和`--exclude`的功能一样，只是所排除的目标是用`<excludefile>`提供的。

### 范围扫描示例

指定一个IP地址然后加一个CIDR的掩码位，如192.168.100.22/24，当然写成192.168.100.0/24也是一样的，因为**nmap需要的是参考IP**。如果扫描的是范围地址，可以192.168.100.1-254这样的书写方式。

```
[root@xuexi ~]# nmap 192.168.100.1/24
 
Starting Nmap 6.40 ( http://nmap.org ) at 2017-06-20 13:22 CST
Nmap scan report for 192.168.100.1
Host is up (0.00053s latency).
Not shown: 992 filtered ports
PORT     STATE SERVICE
21/tcp   open  ftp
135/tcp  open  msrpc
139/tcp  open  netbios-ssn
443/tcp  open  https
445/tcp  open  microsoft-ds
902/tcp  open  iss-realsecure
912/tcp  open  apex-mesh
5357/tcp open  wsdapi
MAC Address: 00:50:56:C0:00:08 (VMware)
 
Nmap scan report for 192.168.100.2
Host is up (0.000018s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
53/tcp open  domain
MAC Address: 00:50:56:E2:16:04 (VMware)

Nmap scan report for 192.168.100.70
Host is up (0.00014s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 00:0C:29:71:81:64 (VMware)

Nmap scan report for 192.168.100.254
Host is up (0.000095s latency).
All 1000 scanned ports on 192.168.100.254 are filtered
MAC Address: 00:50:56:ED:A1:04 (VMware)

Nmap scan report for 192.168.100.62
Host is up (0.0000030s latency).
Not shown: 999 closed ports
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 256 IP addresses (5 hosts up) scanned in 7.96 seconds
```

一般来说，端口全部关闭的很可能不是计算机，而可能是路由器、虚拟网卡等设备。

## 端口状态说明

Nmap功能越来越多，但它赖以成名的是它的核心功能——端口扫描。

Nmap把端口分成六个状态:open(开放的)，closed(关闭的)，filtered(被过滤的)，unfiltered(未被过滤的)，open|filtered(开放或者被过滤的)，或者closed|filtered(关闭或者被过滤的)。

这些状态并非端口本身的性质，而是描述Nmap怎样看待它们。例如，对于同样的目标机器的135/tcp端口，从同网络扫描显示它是开放的，而跨网络做完全相同的扫描则可能显示它是filtered(被过滤的)。

- 1.open：(开放的)用程序正在该端口接收TCP或者UDP报文。它常常是端口扫描的主要目标。
- 2.closed：(关闭的)关闭的端口对于Nmap也是可访问的(它接受Nmap的探测报文并作出响应)，但没有应用程序在其上监听。
- 3.filtered：(被过滤的)由于目标上设置了包过滤(如防火墙设备)，使得探测报文被阻止到达端口，Nmap无法确定该端口是否开放。过滤可能来自专业的防火墙设备，路由器规则或者主机上的软件防火墙。
- 4.unfiltered：(未被过滤的)未被过滤状态意味着端口可访问，但Nmap不能确定它是开放还是关闭。用其它类型的扫描如窗口扫描、SYN扫描、FIN扫描来扫描这些未被过滤的端口可以帮助确定端口是否开放。
- 5.open|filtered：(开放或被过滤的)：当无法确定端口是开放还是被过滤的，Nmap就把该端口划分成这种状态。开放的端口不响应就是一个例子。没有响应也可能意味着目标主机上报文过滤器丢弃了探测报文或者它引发的任何响应。因此Nmap无法确定该端口是开放的还是被过滤的。
- 6.closed|filtered：(关闭或被过滤的)该状态用于Nmap不能确定端口是关闭的还是被过滤的。它只可能出现在IPID Idle扫描中。

## 时间参数优化

改善扫描时间的技术有：忽略非关键的检测、升级最新版本的Nmap(文档中说nmap版本越高性能越好)等。此外，优化时间参数也会带来实质性的优化，这些参数如下：

```
TIMING AND PERFORMANCE:
  ★★★★
  -T<0-5>: Set timing template (higher is faster)
  ★★★★
  --min-hostgroup/max-hostgroup <size>: Parallel host scan group sizes
  ★★★★
  --min-parallelism/max-parallelism <numprobes>: Probe parallelization
  --min-rtt-timeout/max-rtt-timeout/initial-rtt-timeout <time>: Specifies probe round trip time.
  --max-retries <tries>: Caps number of port scan probe retransmissions.
  --host-timeout <time>: Give up on target after this long
  --scan-delay/--max-scan-delay <time>: Adjust delay between probes
  --min-rate <number>: Send packets no slower than <number> per second
  --max-rate <number>: Send packets no faster than <number> per second
```

其中最主要的是前3种：

1. `-T<0-5>`：这表示直接使用namp提供的扫描模板，不同的模板适用于不同的环境下，默认的模板为`-T 3`，具体的看man文档，其实用的很少。

2. `--min-hostgroup <milliseconds>; --max-hostgroup <milliseconds>` (调整并行扫描组的大小)

   Nmap具有并行扫描多主机端口的能力，实现方法是将所有给定的目标IP按空间分成组，然后一次扫描一个组。通常组分的越大效率越高，但分组的缺点是只有当整个组扫描结束后才会返回该组中主机扫描结果。例如，组的大小定义为50，则只有前50个主机扫描结束后才能得到这50个IP内的结果。

   默认方式下，Nmap采取折衷的方法。开始扫描时的组较小，默认值为5，这样便于尽快产生结果，随后增长组的大小，默认最大为1024。但最小和最大确切的值则依赖于所给定的选项。

   `--max-hostgroup`选项用于说明使用最大的组，Nmap不会超出这个大小。`--min-hostgroup`选项说明最小的组，Nmap会保持组大于这个值。如果在指定的接口上没有足够的目标主机来满足所指定的最小值，Nmap可能会采用比所指定的值小的组。

   这些选项的主要用途是说明一个最小组的大小，使得整个扫描更加快速。通常选择256来扫描C类网段，对于端口数较多的扫描，超出该值没有意义，因为它只是分组了，但是cpu资源是有限的。对于端口数较少的扫描，2048或更大的组大小是有帮助的。

3. `--min-parallelism <milliseconds>; --max-parallelism <milliseconds>` (调整探测报文的并行度，即探针数)。

   这些选项用于控制主机组的探测报文数量，可用于**端口扫描和主机发现**。默认状态下，Nmap基于网络性能计算一个理想的并行度，这个值经常改变。如果报文被丢弃，Nmap降低速度，探测报文数量减少。随着网络性能的改善，理想的探测报文数量会缓慢增加。默认状态下，当网络不可靠时，理想的并行度值可能为1，在好的条件下，可能会增长至几百。

   最常见的应用是`--min-parallelism`值大于1，以加快性能不佳的主机或网络的扫描。这个选项具有风险，如果过高则影响准确度，同时也会降低Nmap基于网络条件动态控制并行度的能力。

   一般说来，这个值**要设置的和`--min-hostgroup`的值相等或大于它性能才会提升**。

## 扫描操作系统类型

扫描操作系统。操作系统的扫描有可能会出现误报。

```
C:\Windows\system32>nmap -O 127.0.0.1
 
Starting Nmap 7.40 ( https://nmap.org ) at 2017-03-09 13:18 CST
Nmap scan report for lmlicenses.wip4.adobe.com (127.0.0.1)
Host is up (0.000046s latency).
Not shown: 990 closed ports
PORT      STATE SERVICE
21/tcp    open  ftp
135/tcp   open  msrpc
443/tcp   open  https
445/tcp   open  microsoft-ds
902/tcp   open  iss-realsecure
912/tcp   open  apex-mesh
5357/tcp  open  wsdapi
5678/tcp  open  rrac
10000/tcp open  snet-sensor-mgmt
65000/tcp open  unknown
Device type: general purpose
Running: Microsoft Windows 10
OS CPE: cpe:/o:microsoft:windows_10
OS details: Microsoft Windows 10 1511
Network Distance: 0 hops
 
OS detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 2.33 seconds
```

## 快速扫描存活的主机

要快速扫描存活的主机，需要使用的几个重要选项是：

```
-n：永远不要DNS解析。这个不管是给定地址扫描还是给定网址扫描，加上它速度都会极速提升
-sn：禁止端口扫描
-PE：只根据echo回显判断主机在线，这种类型的选项使用越多，速度越慢，如-PM -PP选项都是类似的，但他们速度要慢的多的多，PE有个缺点，不能穿透防火墙
--min-hostgroup N：当IP太多时，nmap需要分组，然后并扫描，使用该选项可以指定多少个IP一组
--min-parallelism N：这个参数非常关键，为了充分利用系统和网络资源，设置好合理的探针数。一般来说，设置的越大速度越快，且和min-hostgroup的值相等或大于它性能才会提升
```

示例一：扫描192.168.100.0/24网段存活的机器

```
[root@server2 ~]# nmap -sn -n -PE --min-hostgroup 1024 --min-parallelism 1024 192.168.100.1/24

Warning: You specified a highly aggressive --min-hostgroup.
Warning: Your --min-parallelism option is pretty high!  This can hurt reliability.

Starting Nmap 6.40 ( http://nmap.org ) at 2017-06-20 14:30 CST
Nmap scan report for 192.168.100.1
Host is up (0.00036s latency).
MAC Address: 00:50:56:C0:00:08 (VMware)
Nmap scan report for 192.168.100.2
Host is up (0.000051s latency).
MAC Address: 00:50:56:E2:16:04 (VMware)
Nmap scan report for 192.168.100.70
Host is up (0.000060s latency).
MAC Address: 00:0C:29:71:81:64 (VMware)
Nmap scan report for 192.168.100.254
Host is up (0.000069s latency).
MAC Address: 00:50:56:ED:A1:04 (VMware)
Nmap scan report for 192.168.100.62
Host is up.
Nmap done: 256 IP addresses (5 hosts up) scanned in 0.26 seconds
```

255个局域网地址只用了半秒钟。可谓是极速。

再测试扫描下以`www.baidu.com`作为参考地址的地址空间。

```
[root@server2 ~]# nmap -sn -PE -n --min-hostgroup 1024 --min-parallelism 1024 -oX nmap_output.xml www.baidu.com/16

…….省略部分结果

Nmap scan report for 163.177.81.145
Host is up (0.072s latency).
Nmap done: 65536 IP addresses (144 hosts up) scanned in 19.15 seconds
```

可以看到，65535个地址只需19秒就扫描完成了。速度是相当的快。

## 快速扫描端口

既然是扫描端口，就不能使用`-sn`选项，也不能使用`-PE`，否则不会返回端口状态，只会返回哪些主机。

```
[root@server2 ~]# nmap -n -p 20-2000 --min-hostgroup 1024 --min-parallelism 1024 192.168.100.70/24

Warning: You specified a highly aggressive --min-hostgroup.
Warning: Your --min-parallelism option is pretty high!  This can hurt reliability.

Starting Nmap 6.40 ( http://nmap.org ) at 2017-06-20 14:52 CST
Nmap scan report for 192.168.100.1
Host is up (0.00084s latency).
Not shown: 1980 filtered ports
PORT   STATE SERVICE
21/tcp open  ftp
MAC Address: 00:50:56:C0:00:08 (VMware)

Nmap scan report for 192.168.100.2
Host is up (0.000018s latency).
Not shown: 1980 closed ports
PORT   STATE SERVICE
53/tcp open  domain
MAC Address: 00:50:56:E2:16:04 (VMware)
 
Nmap scan report for 192.168.100.70
Host is up (0.000041s latency).
Not shown: 1980 closed ports
PORT   STATE SERVICE
22/tcp open  ssh
MAC Address: 00:0C:29:71:81:64 (VMware)

Nmap scan report for 192.168.100.254
Host is up (0.000035s latency).
All 1981 scanned ports on 192.168.100.254 are filtered
MAC Address: 00:50:56:ED:A1:04 (VMware)

Nmap scan report for 192.168.100.62
Host is up (0.0000020s latency).
Not shown: 1980 closed ports
PORT   STATE SERVICE
22/tcp open  ssh

Nmap done: 256 IP addresses (5 hosts up) scanned in 2.38 seconds
```