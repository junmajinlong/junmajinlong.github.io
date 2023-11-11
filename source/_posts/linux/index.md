---
title: Linux系列文章
p: linux/index.md
date: 2019-07-06 17:37:29
tags: Linux
categories: Linux
---

--------

**[Shell系列文章大纲](/shell/index)**

--------

![](/img/linux/linux_1.jpg)

<a name="basic"></a>

# Linux基础

Linux基础pdf版v3：[Linux基础千锤百炼.pdf](/files/Linux_v3.pdf)

<table><tbody><tr>
<td><ul>
  <li><a href="/linux/linux_file_cmd" target="_blank">1.文件类基础命令</a></li>
  <li>2.Linux系统用户<br>
    <a href="/linux/user_and_group" target="_blank">2.1 系统用户/组管理</a><br>
    <a href="/linux/su_and_sudo" target="_blank">2.2 su和sudo</a><br>
  </li>
  <li><a href="/linux/linux_perm" target="_blank">3.文件的权限管理</a></li>
  <li><a href="/linux/ext_filesystem" target="_blank">4.ext文件系统内部原理机制(精)</a></li>
  <li>5.Linux管理文件系统<br>
    <a href="/linux/fsmgr_about_disk" target="_blank">5.1 了解机械硬盘和硬盘容量计算</a><br>
    <a href="/linux/fsmgr_mkpart_mkfs" target="_blank">5.2 分区和创建文件系统</a><br>
    <a href="/linux/fsmgr_mountfs" target="_blank">5.3 mount挂载各种文件系统</a><br>
    <a href="/linux/fsmgr_swap" target="_blank">5.4 创建并使用swap分区</a><br>
  </li>
</ul></td>
<td><ul>
  <li><a href="/linux/lvm" target="_blank">6.LVM</a></li>
  <li><a href="/linux/raid" target="_blank">7.RAID</a></li>
  <li>8.包管理<br>
    <a href="/linux/pkg_mgr" target="_blank">8.1 学习CentOS的包管理</a><br>
    <a href="/linux/centos8_dnf" target="_blank">8.2 CentOS 8的国内镜像和dnf包管理器</a><br>
  </li>
  <li><a href="/linux/process_and_signal" target="_blank">9.进程和信号(精)</a></li>
  <li><a href="/linux/os_status" target="_blank">10.系统状态统计和查看</a></li>
  <li><a href="/linux/linux_service" target="_blank">11.服务管理</a></li>
  <li><a href="/linux/cron" target="_blank">12.Linux定时任务cron</a></li>
  <li><a href="/linux/linux_network" target="_blank">13.Linux的网络管理</a></li>
  <li><a href="/linux/boot_process_bios_mbr" target="_blank">14.Linux开机详细流程(CentOS 6 BIOS+MBR+SysV)(精)</a> </li>
</ul></td>
</tr></tbody></table>

<a name="blogservice"></a>

# Linux基本服务

<table><tbody><tr>
<td><ul>
<li>1.ssh命令和SSH服务<br>
  1.1 <a href="/linux/ssh" target="_blank">ssh命令和SSH服务详解(精)</a><br>
  1.2 <a href="/linux/ssh_agent" target="_blank">SSH转发代理：ssh-agent用法详解</a><br>
  1.3 <a href="/linux/ssh_port_forward" target="_blank">SSH隧道：端口转发功能详解</a><br>
</li>
<li>2.rsync完全手册<br>
  2.1 <a href="/linux/rsync_basic_usage" target="_blank">rsync(一):基础命令和用法(精)</a><br>
  2.2 <a href="/linux/rsync_inotify_sersync" target="_blank">rsync(二):inotify+rsync和sersync(精)</a><br>
  2.3 <a href="/linux/rsync_algo_analysis" target="_blank">rsync(三):算法原理和工作流程分析(精)</a><br>
  2.4 <a href="/linux/rsync_official_report" target="_blank">rsync(四):翻译：rsync官方推荐技术报告</a><br>
  2.5 <a href="/linux/rsync_how_works" target="_blank">rsync(五):翻译：rsync工作机制(How Rsync Works)</a><br>
  2.6 <a href="/linux/rsync_man" target="_blank">rsync(六):翻译：man rsync(rsync命令中文手册)</a><br>
</li>
<li>3.<a href="/linux/nfs_basic" target="_blank">NFS基本应用</a>
</li>
</ul></td>
<td><ul>
<li>4.<a href="/linux/dhcp" target="_blank">Linux DHCP服务</a></li>
<li>5.无人值守批量安装Linux操作系统<br>
  5.1 <a href="/linux/pxe_centos6" target="_blank">PXE+kickstart安装CentOS 6</a><br>
  5.2 <a href="/linux/kickstart_config" target="_blank">kickstart文件详解</a><br>
  5.3 <a href="/linux/pxe_centos7" target="_blank">PXE+kickstart安装CentOS 7</a><br>
  5.4 <a href="/linux/cobbler" target="_blank">cobbler安装Linux系统</a><br>
</li>
<li>6.数据包过滤和防火墙<br>
  6.1 <a href="/linux/tcpwrapper" target="_blank">tcp_wrapper过滤</a><br>
  6.2 <a href="/linux/iptables" target="_blank">防火墙和iptables</a><br>
  6.3 firewalld(待补充)<br>
</li>
<li>7.<a href="/linux/dns_bind" target="_blank">DNS和bind从基础到深入(精)</a></li>
</ul></td>
</tr></tbody></table>

更多服务软件请移步[网站架构系列](/web_architecture/index)。

<a name="systemd"></a>

# systemd系列

- [前后台进程、孤儿进程和daemon类进程的父子关系](/linux/process_relationship)  
- [systemd时代的服务管理](/linux/systemd/service_manage)  
- [systemd service之：服务配置文件编写(1)](/linux/systemd/service_1)  
- [systemd service之：服务配置文件编写(2)](/linux/systemd/service_2)  
- [systemd时代的开机自启动任务](/linux/systemd/auto_tasks_on_boot)  
- [systemd时代的运行级别](/linux/systemd/runlevel)  
- [systemd时代的/etc/fstab](/linux/systemd/systemd_fstab)  
- [systemd timesyncd做时间同步](/linux/systemd/systemd_timesyncd)  
- [systemd timer：取代cron和at的定时任务](/linux/systemd/systemd_timer)  
- [systemd path：实时监控文件和目录的变动](/linux/systemd/systemd_path)  
- [systemd时代的开机启动流程(GPT+systemd)](/linux/systemd/systemd_bootup)  

<a name="blogopenssl"></a>

# openssl系列

- [加密、签名和SSL握手机制细节(精)](/linux/ssl_details)
- [openssl命令用法详解](/linux/openssl_subcmds)
- [openssl申请证书、颁发证书、自建CA](/linux/openssl_ca)
- [SSL官方书籍下载](https://junmajinlong.lanzouj.com/iW6t31e7tzcj)


<a name="others"></a>

# 杂项内容

<table>
<tbody>
<tr>
<td>
<ul>
<li><a href="/linux/cpio_usage" target="_blank">cpio归档工具用法详细说明</a></li>
<li><a href="/linux/receive_x" target="_blank">使用xmanager接收图形界面</a></li>
<li><a href="/linux/tcpdump_basic_usage" target="_blank">tcpdump抓包工具基本用法</a></li>
<li><a href="/linux/nmap_usage" target="_blank">nmap网络扫描工具基本用法</a></li>
<li><a href="/linux/linux_hot_disk" target="_blank">Linux上磁盘热插拔</a></li>
<li><a href="/linux/linux_tty" target="_blank">Linux终端类型</a></li>
<li><a href="/linux/terminal_share" target="_blank">Linux终端会话实时共享(kibitz)</a></li>
<li><a href="/linux/terminal_share" target="_blank">Linux录制、回放和实时共享终端操作</a></li>
<li><a href="/linux/centos_repos" target="_blank">为CentOS配置常用软件源：epel和IUS</a></li>
<li><a href="/linux/os_hostname" target="_blank">CentOS 7主机名的弯弯绕绕</a></li>
<li><a href="/linux/why_one_prefix" target="_blank">绝对路径为什么是"/usr"而不是"//usr"</a></li>
<li><a href="/linux/euid_ruid" target="_blank">理解Effective UID(EUID)和Real UID(RUID)</a></li>
</ul>
</td>
<td>
<ul>
<li><a href="/linux/md5sum" target="_blank">Linux中文件MD5校验</a></li>
<li><a href="/linux/make_shadow_passwd" target="_blank">手动生成/etc/shadow文件中的密码</a></li>
<li><a href="/linux/wget_usage" target="_blank">wget命令常用选项和示例</a></li>
<li><a href="/linux/ports_busy" target="_blank">Linux查询端口是否被占用的几种方法</a></li>
<li><a href="/linux/yum_install_php_errors" target="_blank">yum安装新版php遇到的坑</a></li>
<li><a href="/linux/du_df" target="_blank">详细分析du和df统计结果为什么不一样</a></li>
<li><a href="/linux/sshfs" target="_blank">sshfs基于ssh挂载远程目录</a></li>
<li><a href="/linux/dir_diff" target="_blank">Linux下快速比较两个目录的不同</a></li>
<li><a href="/coding/linux_file_type" target="_blank">搞懂Linux下的几种文件类型</a></li>
<li><a href="/linux/data_to_remote" target="_blank">Linux将不断生成的新数据发送到远程节点</a></li>
<li><a href="/linux/mount_bind" target="_blank">mount bind功能详解</a></li>
</ul>
</td>
</tr>
</tbody>
</table>


<a name="mytranslations"></a>
# 我的个人翻译

- [翻译：grub2详解(翻译和整理官方手册)](/linux/grub2)  
- [翻译：Bios boot partition](/linux/bios_boot_partition_wiki_translate)
- [翻译：man ssh(ssh命令中文手册)](/linux/man_ssh_translate)  
- [翻译：rsync官方推荐技术报告](/linux/rsync_official_report)
- [翻译：rsync工作机制(How Rsync Works)](/linux/rsync_how_works)
- [翻译：man rsync(rsync命令中文手册)](/linux/rsync_man)
- [翻译：info sort(sort命令中文手册)](/shell/sort_trans)
- [翻译：info grep(grep命令中文手册)](/shell/grep_translate)  
- [翻译：info sed(sed命令中文手册 + 注解)](/shell/sed2)
- [翻译：man getopt(1)中文手册](/shell/getopt_translate)