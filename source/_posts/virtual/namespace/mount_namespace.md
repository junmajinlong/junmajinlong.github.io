---
title: Linux namespace之：mount namespace
p: virtual/namespace/mount_namespace.md
date: 2020-10-10 17:37:31
tags: Virtualization
categories: Virtualization
---

--------
**目录**：  
- **[Linux namespace概述](/virtual/namespace/ns_overview)**  
- **[Linux namespace之：uts namespace](/virtual/namespace/uts_namespace)**  
- **[Linux namespace之：mount namespace](/virtual/namespace/mount_namespace)**  
- **[Linux namespace之：pid namespace](/virtual/namespace/pid_namespace)**  
- **[Linux namespace之：network namespace](/virtual/namespace/network_namespace)**  
- **[Linux namespace之：user namespace](/virtual/namespace/user_namespace)**  

--------

# 理解mount namespace

用户通常使用mount命令来挂载普通文件系统，但实际上mount能挂载的东西非常多，甚至连现在功能完善的Linux系统，其内核的正常运行也都依赖于挂载功能，比如挂载根文件系统`/`。其实所有的挂载功能和挂载信息都由内核负责提供和维护，mount命令只是发起了mount()系统调用去请求内核。

mount namespace可隔离出一个具有独立挂载点信息的运行环境，内核知道如何去维护每个namespace的挂载点列表。即**每个namespace之间的挂载点列表是独立的，各自挂载互不影响**。

内核将每个进程的挂载点信息保存在`/proc/<pid>/{mountinfo,mounts,mountstats}`三个文件中：

```bash
$ ls -1 /proc/$$/mount*
/proc/26276/mountinfo
/proc/26276/mounts
/proc/26276/mountstats
```

**具有独立的挂载点信息，意味着每个mnt namespace可具有独立的目录层次**，这在容器中起了很大作用：容器可以挂载只属于自己的文件系统。

当创建mount namespace时，内核将拷贝一份当前namespace的挂载点信息列表到新的mnt namespace中，此后两个mnt namespace就没有了任何关系(不是真的毫无关系，参考后文shared subtrees)。

创建mount namespace的方式是使用unshare命令的`--mount, -m`选项：

```bash
# 创建mount+uts namespace
# -m或--mount表示创建mount namespace
# 可同时创建具有多种namespace类型的namespace
unshare --mount --uts <program>
```

下面做一个简单的试验，在root namespace中挂载1.iso文件到/mnt/iso1目录，在新建的mount+uts namespace中挂载2.iso到/mnt/iso2目录：

```bash
[~]->$ cd
[~]->$ mkdir iso
[~]->$ cd iso
[iso]->$ mkdir -p iso1/dir1
[iso]->$ mkdir -p iso2/dir2  
[iso]->$ mkisofs -o 1.iso iso1  # 将iso1目录制作成镜像文件1.iso
[iso]->$ mkisofs -o 2.iso iso2  # 将iso2目录制作成镜像文件2.iso
[iso]->$ ls
1.iso  2.iso  iso1  iso2
[iso]->$ sudo mkdir /mnt/{iso1,iso2}
[iso]->$ ls -l /proc/$$/ns/mnt
lrwxrwxrwx 1 ... /proc/26276/ns/mnt -> 'mnt:[4026531840]'

# 在root namespace中挂载1.iso到/mnt/iso1目录
[iso]->$ sudo mount 1.iso /mnt/iso1  
mount: /mnt/iso: WARNING: device write-protected, mounted read-only.
[iso]->$ mount | grep iso1
/home/longshuai/iso/1.iso on /mnt/iso1 type iso9660

# 创建mount+uts namespace
[iso]->$ sudo unshare -m -u /bin/bash
# 虽然这个namespace是mount+uts的namespace
# 但注意mnt namespace和uts namespace的inode并不一样
root@longshuai-vm:/home/longshuai/iso# ls -l /proc/$$/ns
lrwxrwxrwx ... cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx ... ipc -> 'ipc:[4026531839]'
lrwxrwxrwx ... mnt -> 'mnt:[4026532588]'
lrwxrwxrwx ... net -> 'net:[4026531992]'
lrwxrwxrwx ... pid -> 'pid:[4026531836]'
lrwxrwxrwx ... pid_for_children -> 'pid:[4026531836]'
lrwxrwxrwx ... user -> 'user:[4026531837]'
lrwxrwxrwx ... uts -> 'uts:[4026532589]'

# 修改主机名为ns1
root@longshuai-vm:/home/longshuai/iso# hostname ns1
root@longshuai-vm:/home/longshuai/iso# exec $SHELL

# 在namespace中，可以看到root namespace中的挂载信息
root@ns1:/home/longshuai/iso# mount | grep 'iso1' 
/home/longshuai/iso/1.iso1 on /mnt/iso1 type iso9660

# namespace中挂载2.iso2
root@ns1:/home/longshuai/iso# mount 2.iso2 /mnt/iso2/
mount: /mnt/iso2: WARNING: device write-protected, mounted read-only.
root@ns1:/home/longshuai/iso# mount | grep 'iso[12]'
/home/longshuai/iso/1.iso1 on /mnt/iso1 type iso9660
/home/longshuai/iso/2.iso2 on /mnt/iso2 type iso9660

# 在namespace中卸载iso1
root@ns1:/home/longshuai/iso# umount /mnt/iso1/
root@ns1:/home/longshuai/iso# mount | grep 'iso[12]' 
/home/longshuai/iso/2.iso2 on /mnt/iso2 type iso9660
root@ns1:/home/longshuai/iso# ls /mnt/iso1/
root@ns1:/home/longshuai/iso# ls /mnt/iso2
dir2

#### 打开另一个Shell终端窗口
# iso1挂载仍然存在，且没有iso2的挂载信息
[iso]->$ mount | grep iso
/home/longshuai/iso/1.iso1 on /mnt/iso1 type iso9660
[iso]->$ ls /mnt/iso2
[iso]->$ ls /mnt/iso1
dir1
```

以上是mount namespace的基本内容，只有一个关键点：创建mnt namespace时会拷贝当前namespace的挂载点信息，之后两个namespace就没有关系了。

## mnt namespace: shared subtrees

Linux的每个挂载点都具有一个决定该挂载点**是否共享子挂载点**的属性，称为shared subtrees。该属性以决定某挂载点之下新增或移除子挂载点时，是否同步影响它【副本】挂载点。如不了解该属性的作用，参考[mount bind和shared subtrees详解](https://www.junmajinlong.com/linux/mount_bind#shared-subtrees)。

简单说说shared subtrees特性，该特性有什么用呢？以mnt namespace为例来简单介绍一下shared subtrees特性。

假设基于root namespace创建了一个mnt namespace(ns1)，那么ns1将具有当前root namespace的挂载点信息拷贝。如果此时新插入了一块磁盘并对其分区格式化，然后在root namespace中对其进行挂载，默认情况下，在ns1中将看不到新挂载的文件系统。这种默认行为可以通过修改shared subtrees属性来改变。

其实，用户创建namespace，其目的一般是希望创建完全隔离的运行环境，所以默认情况下，拷贝挂载点信息时不会拷贝shared subtrees属性，而是将mount namespace中的所有挂载点的shared subtrees属性设置为private。

所以，假如namespace ns1中/mnt/foo是一个挂载点目录，基于ns1创建了一个mnt namespace ns2，在默认情况下：

- 如果此时在ns1中新增一个挂载点/mnt/foo/bar，将不会影响到ns2中的/mnt/foo  
- 如果此时在ns2中新增一个挂载点/mnt/foo/baz，也不会影响到ns1中的/mnt/foo  
- 移除挂载点操作也一样  

但这种默认行为可以改变。

`unshare`有一个选项`--propagation private|shared|slave|unchanged`可控制创建mnt namespace时挂载点的共享方式。

- private：表示新创建的mnt namespace中的挂载点的shared subtrees属性都设置为private，即ns1和ns2的挂载点互不影响  
- shared：表示新创建的mnt namespace中的挂载点的shared subtrees属性都设置为shared，即ns1或ns2中新增或移除子挂载点都会同步到另一方  
- slave：表示新创建的mnt namespace中的挂载点的shared subtrees属性都设置为slave，即ns1中新增或移除子挂载点会影响ns2，但ns2不会影响ns1  
- unchanged：表示拷贝挂载点信息时也拷贝挂载点的shared subtrees属性，也就是说挂载点A原来是shared，在mnt namespace中也将是shared  
- 不指定`--progapation`选项时，创建的mount namespace中的挂载点的shared subtrees默认值是private  

例如：

```bash
# root namespace：ns0
$ sudo mount --bind foo bar
$ sudo mount --make-shared bar    # 挂载点设置为shared

# 创建mnt namespace：ns1，且也拷贝shared属性
$ PS1="ns1$ " sudo unshare -m -u --propagation unchanged sh
[ns1]$ grep 'foo' /proc/self/mountinfo 
944 682 8:5 foo bar rw,relatime shared:1

# 在ns1中bar挂载点下新增子挂载点，子挂载点将同步到ns0
# 因为foo和bar已绑定，且bar的属性是shared，所以还会同步到foo目录下
[ns1]$ sudo mount --bind baz bar/subfoo
[ns1]$ tree foo bar 
foo
└── subfoo
    └── subbaz
bar
└── subfoo
    └── subbaz
[ns1]$ grep 'foo' /proc/self/mountinfo
944 682 8:5 foo bar rw,relatime shared:1
945 944 8:5 baz bar/subfoo rw,relatime shared:1
947 682 8:5 baz foo/subfoo rw,relatime shared:1

# 第二个窗口会话中查看ns0的挂载点信息
# 已经同步过来了
$ grep 'foo' /proc/self/mountinfo
622 29 8:5  foo bar rw,relatime shared:1
948 622 8:5 baz bar/subfoo rw,relatime shared:1
946 29 8:5  baz foo/subfoo rw,relatime shared:1
$ tree foo bar 
foo
└── subfoo
    └── subbaz
bar
└── subfoo
    └── subbaz
```

