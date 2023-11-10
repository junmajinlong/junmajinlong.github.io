---
title: MySQL系列文章
p: mysql/index.md
date: 2019-07-06 17:37:29
tags: MySQL
categories: MySQL
---


# 1.MySQL/MariaDB 语法和基础篇

[MySQL 基本语法：和 SQL Server 语法的差异小归纳](http://www.cnblogs.com/f-ck-need-u/p/7592501.html)  
[数据类型详解和存储机制](http://www.cnblogs.com/f-ck-need-u/p/7729251.html) (精)  
[内置函数大全](http://www.cnblogs.com/f-ck-need-u/p/7740235.html)  
[变量](http://www.cnblogs.com/f-ck-need-u/p/8695767.html)  
[DML(1)：数据插入](http://www.cnblogs.com/f-ck-need-u/p/8907617.html)  
[DML(2)：数据更新、删除](http://www.cnblogs.com/f-ck-need-u/p/8912026.html)  
[开窗函数用法说明](http://www.cnblogs.com/f-ck-need-u/p/8945248.html)  
[存储过程和函数](http://www.cnblogs.com/f-ck-need-u/p/8710324.html)  
[游标](http://www.cnblogs.com/f-ck-need-u/p/8722244.html)  
[流程控制语句](http://www.cnblogs.com/f-ck-need-u/p/8724063.html)  
[触发器](http://www.cnblogs.com/f-ck-need-u/p/8870446.html)  
表表达式 (1)：派生表 (待补充)  
[表表达式 (2)：CTE(公用表表达式)](http://www.cnblogs.com/f-ck-need-u/p/8875863.html)  
[表表达式 (3)：视图](http://www.cnblogs.com/f-ck-need-u/p/8870908.html)  

## 杂项

[SQL 语句逻辑执行过程 (MSSQL/MYSQL/MariaDB)](http://www.cnblogs.com/f-ck-need-u/p/8656828.html)  
[从关系模型和集合的角度分析 SQL 相关语法 (适用于 MSSQL/MYSQL/MariaDB)](http://www.cnblogs.com/f-ck-need-u/p/8656828.html)  
[从集合的无序性看待关系型数据库中的 "序"(适用于 MSSQL/MYSQL/MariaDB)](http://www.cnblogs.com/f-ck-need-u/p/8718662.html)  
[MyISAM 存储引擎读、写操作的优先级](http://www.cnblogs.com/f-ck-need-u/p/8907252.html)  
[Can't connect to local MySQL server through socket](https://www.cnblogs.com/f-ck-need-u/p/9098664.html)  
[You must reset your password using ALTER USER statement before executing this statement.](https://www.cnblogs.com/f-ck-need-u/p/9221317.html)  

更多文章待续...

# 2.MySQL/MariaDB 管理篇

[MySQL、MariaDB 安装和多实例配置](http://www.cnblogs.com/f-ck-need-u/p/7590376.html)  
[用户和权限管理](http://www.cnblogs.com/f-ck-need-u/p/8994220.html)  
[锁](http://www.cnblogs.com/f-ck-need-u/p/8995475.html)  
[事务和事务隔离级别](http://www.cnblogs.com/f-ck-need-u/p/8997814.html)  
[日志 (一)](http://www.cnblogs.com/f-ck-need-u/p/9001061.html)  
[日志 (二)：事务日志 (redo log 和 undo log)](http://www.cnblogs.com/f-ck-need-u/p/9010872.html)  
[备份和恢复 (一)：mysqldump 工具用法详述](http://www.cnblogs.com/f-ck-need-u/p/9013458.html)  
[备份和恢复 (二)：导入、导出表数据](http://www.cnblogs.com/f-ck-need-u/p/9013643.html)  
[备份和恢复 (三)：xtrabackup 用法和原理详述](http://www.cnblogs.com/f-ck-need-u/p/9018716.html)  
[MySQL 复制 (一)：复制理论和传统复制的配置](https://www.cnblogs.com/f-ck-need-u/p/9155003.html)  
[MySQL 复制 (二)：基于 GTID 复制](https://www.cnblogs.com/f-ck-need-u/p/9164823.html)  
[MySQL 复制 (三)：半同步复制](https://www.cnblogs.com/f-ck-need-u/p/9166452.html)  

[数据库性能测试：sysbench 用法详解](https://www.cnblogs.com/f-ck-need-u/p/9279703.html)

[show processlist 说明](http://www.cnblogs.com/f-ck-need-u/p/7742153.html)

更多文章待续...

# 3.MySQL 集群

[MySQL 集群结构说明](https://www.cnblogs.com/f-ck-need-u/p/9278900.html) (必看)  
[sharding：谁都能读懂的分库、分表、分区](https://www.cnblogs.com/f-ck-need-u/p/9388407.html)  

## 3.1 MySQL 高可用：组复制、Galera

[MySQL 组复制官方手册翻译. rar(下载)](https://files.cnblogs.com/files/f-ck-need-u/MySQL_GR.rar)

[1.MySQL 高可用之组复制 (1)：组复制技术简介](https://www.cnblogs.com/f-ck-need-u/p/9216828.html)  
[2.MySQL 高可用之组复制 (2)：配置单主模型的组复制](https://www.cnblogs.com/f-ck-need-u/p/9203154.html)  
[3.MySQL 高可用之组复制 (3)：配置多主模型的组复制](https://www.cnblogs.com/f-ck-need-u/p/9215013.html)  
[4.MySQL 高可用之组复制 (4)：组复制理论透彻分析](https://www.cnblogs.com/f-ck-need-u/p/9225442.html)  
[5. 翻译：MySQL 组复制的限制和局限性](https://www.cnblogs.com/f-ck-need-u/p/9197442.html)  
[6. 翻译：监控 MySQL 组复制](https://www.cnblogs.com/f-ck-need-u/p/9204774.html)

MGR 文章推荐: https://sq.163yun.com/blog/article/223220453915172864

[1.PXC 快速入门：搭建 PXC](https://www.cnblogs.com/f-ck-need-u/p/9364877.html)  

## 3.2 MySQL 中间件：ProxySQL

[中间件入门：MySQL Router 实现 MySQL 的读写分离](https://www.cnblogs.com/f-ck-need-u/p/9276639.html)

[ProxySQL 中文手册 (官方文档翻译).rar(下载)](https://files.cnblogs.com/files/f-ck-need-u/ProxySQL_cn.rar)

[1.ProxySQL(1)：简介和安装](https://www.cnblogs.com/f-ck-need-u/p/9278818.html)  
[2.ProxySQL(2)：初试读写分离](https://www.cnblogs.com/f-ck-need-u/p/9278839.html)  
[3.ProxySQL(3)：Admin 管理接口](https://www.cnblogs.com/f-ck-need-u/p/9281199.html)  
[4.ProxySQL(4)：多层配置系统](https://www.cnblogs.com/f-ck-need-u/p/9280793.html)  
[5.ProxySQL(5)：线程、线程池、连接池](https://www.cnblogs.com/f-ck-need-u/p/9281909.html)  
[6.ProxySQL(6)：管理后端节点](https://www.cnblogs.com/f-ck-need-u/p/9286922.html)  
[7.ProxySQL(7)：路由规则详述](https://www.cnblogs.com/f-ck-need-u/p/9300829.html)  
[8.ProxySQL(8)：SQL 语句的重写规则](https://www.cnblogs.com/f-ck-need-u/p/9309760.html)  
[9.ProxySQL(9)：查询缓存功能](https://www.cnblogs.com/f-ck-need-u/p/9314459.html)  
[10.ProxySQL(10)：读写分离方法论](https://www.cnblogs.com/f-ck-need-u/p/9318558.html)  
[11.ProxySQL(11)：链式规则 ( flagIN 和 flagOUT )](https://www.cnblogs.com/f-ck-need-u/p/9350631.html)  
[12.ProxySQL(12)：禁止多路路由](https://www.cnblogs.com/f-ck-need-u/p/9372447.html)  
[13.ProxySQL(13)：ProxySQL 集群](https://www.cnblogs.com/f-ck-need-u/p/9362822.html)  
[14.ProxySQL(14)：ProxySQL+PXC](https://www.cnblogs.com/f-ck-need-u/p/9372382.html)  
[15.ProxySQL(15)：ProxySQL + 组复制](https://www.cnblogs.com/f-ck-need-u/p/9383126.html)  
下面这篇待写  
[16.ProxySQL(16)：ProxySQL 实现 sharding](https://www.cnblogs.com/f-ck-need-u/p/7586194.html)  

# 4. 我的译作 (39)

[MySQL 组复制官方手册翻译](https://files.cnblogs.com/files/f-ck-need-u/MySQL_GR.rar)  
[ProxySQL 中文手册 (官方文档翻译)](https://github.com/malongshuai/proxysql/wiki)  
[MariaDB 官方手册翻译集合](https://www.cnblogs.com/f-ck-need-u/p/10697909.html)

更多文章待续...



