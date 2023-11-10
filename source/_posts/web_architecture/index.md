---
title: 网站架构系列
p: web_architecture/index.md
date: 2019-07-06 17:37:29
tags: Web_architecture
categories: Web_architecture
---

# 重要的背景知识

- [1. 零复制 (zero copy) 技术](/coding/zero_copy)  
- [2. 五种 IO 模型分析](/coding/IO_Model)(精)  
- [3. 不可不知的 socket 和 TCP 连接过程](/coding/tcp_socket)(精)  
- [4. 简单说明 CGI 和动态请求是什么](https://www.cnblogs.com/f-ck-need-u/p/7627035.html)(精)  
- [5. 正向代理、透明代理、反向代理的区别说明](https://www.cnblogs.com/f-ck-need-u/p/9739870.html)  
- [6. 存储基础知识](https://www.cnblogs.com/f-ck-need-u/p/9069823.html)  

更多文章待续...

---

# 架构入门

<a name="httpd"></a>

## Web 服务:apache httpd

- 1.[httpd 配置文件规则说明和基本指令](http://www.cnblogs.com/f-ck-need-u/p/7636836.html)  
- 2.[httpd 轻松配置虚拟主机](http://www.cnblogs.com/f-ck-need-u/p/7632878.html)  
- 3.[httpd 网页身份认证](http://www.cnblogs.com/f-ck-need-u/p/7634205.html)  
- 4.[httpd 日志和日志轮替工具](http://www.cnblogs.com/f-ck-need-u/p/7635409.html)  
- 5.[httpd 路径映射和重定向](http://www.cnblogs.com/f-ck-need-u/p/7634381.html)  
- 6.[httpd 三种 MPM 的原理剖析](http://www.cnblogs.com/f-ck-need-u/p/7628728.html) (精)  
- 7.[httpd 反向代理用法指南](http://www.cnblogs.com/f-ck-need-u/p/7651234.html)  
- 8.[httpd 编译细节](http://www.cnblogs.com/f-ck-need-u/p/7605563.html) (精)  
- 9.[搭建 LAMP 环境示例](http://www.cnblogs.com/f-ck-need-u/p/7642992.html)  
- 10.[httpd 添加新模块](http://www.cnblogs.com/f-ck-need-u/p/8413455.html)  
- 11.[httpd htpasswd 命令](http://www.cnblogs.com/f-ck-need-u/p/8413490.html)  

<a name="nginx"></a>

## Web服务：Nginx

- 1.[nginx 基础及提供 web 服务 (nginx.conf 详解)](http://www.cnblogs.com/f-ck-need-u/p/7683027.html)  
- 2.[nginx 版本平稳切换](http://www.cnblogs.com/f-ck-need-u/p/7658111.html)  
- 3.[nginx 的反向代理功能和缓存功能](http://www.cnblogs.com/f-ck-need-u/p/7684732.html) (精)  
- 4.[nginx URL 重写](http://www.cnblogs.com/f-ck-need-u/p/7685485.html)  
- 5.[LNMP 之 nginx+php-fpm(两种通信方式)](http://www.cnblogs.com/f-ck-need-u/p/7657493.html)  

<a name="nginx"></a>

## Web服务：Tomcat 

- 1.[背景知识和 tomcat 安装](http://www.cnblogs.com/f-ck-need-u/p/7717488.html)  
- 2.[tomcat 配置文件详解和部署简介](http://www.cnblogs.com/f-ck-need-u/p/8120008.html) (精)  
- 3.[tomcat 处理连接的详细过程](http://www.cnblogs.com/f-ck-need-u/p/8408670.html) (精)  
- 4.[tomcat 图形管理和身份认证](http://www.cnblogs.com/f-ck-need-u/p/8409723.html)  
- 5.[nginx/httpd+tomcat 及负载均衡 tomcat](http://www.cnblogs.com/f-ck-need-u/p/8414043.html)(精)  

<a name="cluster"></a>

# 架构初步

## 漫谈

- 漫谈负载均衡 (待补充)
- 网站伸缩性架构 (待补充)
- 漫谈高可用 (待补充)
- 网站高可用架构 (待补充)
- 网站架构中的缓存 (待补充)

## 负载均衡 (反向代理)、高可用、缓存

<a name="lvs"></a>

### LVS+KeepAlived

- 1.[LVS(一)：基本概念和三种模式](https://www.cnblogs.com/f-ck-need-u/p/8451982.html)  
- 2.[LVS(二)：VS_TUN 和 VS_DR 的 arp 问题](https://www.cnblogs.com/f-ck-need-u/p/8455004.html) (精)  
- 3.[LVS(三)：ipvsadm 命令](https://www.cnblogs.com/f-ck-need-u/p/8527125.html)  
- 4.[LVS(四)：详细剖析 VS/NAT 和 VS/DR](https://www.cnblogs.com/f-ck-need-u/p/8472744.html)(精)  
- 5.[LVS(五)：lvs 和 nginx 的 wrr 算法规律分析](https://www.cnblogs.com/f-ck-need-u/p/9490629.html) (精)  
- 6.[KeepAlived(一)：基本概念和配置文件](https://www.cnblogs.com/f-ck-need-u/p/8483807.html)  
- 7.[KeepAlived(二)：keepalived+lvs](https://www.cnblogs.com/f-ck-need-u/p/8492298.html)  
- 8.[KeepAlived(三)：vrrp 故障转移 (+haproxy)](https://www.cnblogs.com/f-ck-need-u/p/8566233.html)  

<a name="haproxy"></a>

### 强势反代 HAProxy

- 1.[安装 haproxy 和 haproxy 命令](http://www.cnblogs.com/f-ck-need-u/p/8546010.html)  
- 2.[haproxy 的丰富特性简介](http://www.cnblogs.com/f-ck-need-u/p/8545723.html) (精)  
- 3.[haproxy 配置示例和需要考虑的问题](http://www.cnblogs.com/f-ck-need-u/p/8540805.html) (精)  
- 4.[haproxy 配置文件详解和 ACL](http://www.cnblogs.com/f-ck-need-u/p/8502593.html)  
- 5.[haproxy 实现会话保持 (1):cookie](http://www.cnblogs.com/f-ck-need-u/p/8553190.html)(精)  
- 6.[haproxy 实现会话保持 (2):stick table](http://www.cnblogs.com/f-ck-need-u/p/8558514.html)(精)  
- 7.[haproxy 高效的 stick table 复制功能](http://www.cnblogs.com/f-ck-need-u/p/8565998.html)  
- 8.[haproxy 代理 MySQL 要考虑的问题](https://www.cnblogs.com/f-ck-need-u/p/9370579.html)  

<a name="ha"></a>

### 高可用 + drbd

**下载：** [PaceMaker 入门指南. pdf](https://download.csdn.net/download/a905815661/10295871)

- 1.heartbeat/corosync、pacemaker 之间的关系  
- 2.[Resource Agent:LSB 和 OCF](http://www.cnblogs.com/f-ck-need-u/p/8724402.html)  
- 3.[heartbeat 单独提供高可用服务](http://www.cnblogs.com/f-ck-need-u/p/8587882.html)(精)  
- 4.[drbd(一)：简介、同步机制和安装](http://www.cnblogs.com/f-ck-need-u/p/8673178.html)  
- 5.[drbd(二)：配置和使用 drbd](http://www.cnblogs.com/f-ck-need-u/p/8678883.html)  
- 6.[drbd(三)：drbd 的状态说明](http://www.cnblogs.com/f-ck-need-u/p/8684648.html)  
- 6.[drbd(四)：drbd 多节点 (drbd9)](http://www.cnblogs.com/f-ck-need-u/p/8691373.html)  
- 7.heartbeat+drbd+nfs 最佳实践  
- 8.pacemaker+corosync 提供高可用服务  

<a name="cache"></a>

### 缓存 (待续......)

* * *

# 工具类

<a name="ansible"></a>

## Ansible系列：一步到位玩透Ansible(原51cto专栏)

这些是我之前写在51cto专栏的Ansible文章，是从0到1玩透性质的，循序渐进且非常系统性，大概39W字，转成pdf有430多页，现在免费分享出来(已请求51cto下架专栏)。

**《一步到位玩透Ansible》大纲**：  

- [1.学习不迷茫：Ansible要如何学至精通](/ansible/index)  
- [2.初入Ansible世界：用法概览和初体验](/ansible/index)  
- [3.制定演员表：inventory](/ansible/index)  
- [4.嘿，瞧瞧Ansible的灵魂：playbook](/ansible/index)  
- [5.Ansible力量初显：批量初始化服务器](/ansible/index)  
- [6.更大的舞台(1)：组织多个文件以及Role](/ansible/index)  
- [7.更大的舞台(2)：利用Role部署LNMP案例](/ansible/index)  
- [8.回归Ansible并进阶：变量、条件、循环、异常处理及其它](/ansible/index)  
- [9.如虎添翼的力量：解锁强大的Jinja2模板](/ansible/index)  
- [10.服务0 downtime的追求：Haproxy+Nginx集群服务的滚动发布和节点伸缩](/ansible/index)  
- [11.Ansible你快点：Ansible执行过程分析、异步、效率优化](/ansible/index)  
- [12.让Ansible更安全：使用Vault进行加密](/ansible/index)  
- [13.蚂蚁多了也咬不死Ansible：Ansible Tower](/ansible/index)  
- [14.Ansible管理docker和openstack](/ansible/index)  
- [15.意外之喜：Ansible管理Windows主机](/ansible/index)  
- [16.成就感源于创造：自己动手写Ansible模块](/ansible/index)  

<a name="zk"></a>

##  ZooKeeper系列

- 1.[翻译：ZooKeeper OverView](https://www.cnblogs.com/f-ck-need-u/p/9231153.html)  
- 2.[ZooKeeper系列(1)：安装搭建ZooKeeper环境](https://www.cnblogs.com/f-ck-need-u/p/9235308.html)  
- 3.[ZooKeeper系列(2)：命令行工具zkCli.sh](https://www.cnblogs.com/f-ck-need-u/p/9232829.html)  
- 4.[ZooKeeper系列(3)：znode说明和znode状态](https://www.cnblogs.com/f-ck-need-u/p/9233249.html)  
- 5.[ZooKeeper系列(4)：ZK的配置文件详解](https://www.cnblogs.com/f-ck-need-u/p/9236200.html)  
- 6.[ZooKeeper系列(5)：ZK的日志和快照](https://www.cnblogs.com/f-ck-need-u/p/9236954.html)  
- 7.[ZooKeeper系列(6)：ZK的伸缩性和Observer角色](https://www.cnblogs.com/f-ck-need-u/p/9238123.html)  

# openvpn

- [应用OpenVPN](/web_architecture/openvpn)  

# Linux上配置使用iSCSI

- [Linux 上配置使用 iSCSI 详细说明](http://www.cnblogs.com/f-ck-need-u/p/9067906.html)  




