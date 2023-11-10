---
title: mount bind功能详解
p: linux/mount_bind.md
date: 2020-10-06 13:20:42
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# mount bind功能详解

## mount bind用法

mount bind可为当前挂载点绑定一个新的挂载点。

执行如下命令，可创建foo目录的一个镜像目录bar，它们已经绑定在一起：

```bash
mkdir foo bar
mount --bind foo bar
```

mount bind绑定后的两个目录类似于硬链接，无论读写bar还是读写foo，都会反应在另一方，内核在底层所操作的都是同一个物理位置。

```bash
echo 1 >foo/a.txt
cat bar/a.txt      # 输出1
```

将bar卸载后，bar目录回归原始空目录状态，期间所执行的修改都保留在foo目录下：

```bash
$ sudo umount bar
$ ls foo
a.txt
$ ls bar    # 空
```

mount bind除了可以绑定两个普通目录，还可以绑定挂载点。

假设/mnt/foo是一个挂载点，执行如下命令：

```bash
mount --bind /mnt/foo /mnt/bar
```

这将使得/mnt/bar成为/mnt/foo的一个镜像挂载点，读写/mnt/bar和读写/mnt/foo是等价的，且无论卸载哪一方，另一方都依旧可用。

## shared subtrees

对于挂载点/mnt/foo，执行如下命令：

```bash
mount --bind /mnt/foo /mnt/bar
```

bind绑定了这两个挂载点/mnt/foo和/mnt/bar，使得操作/mnt/foo和操作/mnt/bar的效果是一样的。

但结论不完全如此。

如果不是简单的读写这两个挂载点，而是在这两个挂载点下新增或移除子挂载点，结论还有待进一步商榷。

Linux的每个挂载点都具有一个决定该挂载点**是否共享子挂载点**的属性，称为**shared subtrees**(注：这是挂载点的属性，就像是否只读一样的属性)。该属性用于决定**某挂载点之下新增或移除子挂载点时，是否同步影响bind另一端的挂载点**。

shared subtrees有四种属性值，它们的设置方式分别为：

```bash
# 挂载前直接设置shared subtrees属性
mount --make-private    --bind <olddir> <newdir>
mount --make-shared     --bind <olddir> <newdir>
mount --make-slave      --bind <olddir> <newdir>
mount --make-unbindable --bind <olddir> <newdir>

# 或者挂载后设置挂载点属性
mount --bind <olddir> <newdir>
mount --make-private    <newdir>
mount --make-shared     <newdir> 
mount --make-slave      <newdir>
mount --make-unbindable <newdir> 
```

对于shared subtrees这几种属性值，以`mount --bind foo bar`为例：  

- `private`属性：表示在foo或bar下新增、移除子挂载点，不会体现在另一方，即`foo <-x-> bar`  
- `shared`属性：表示在foo或bar下新增、移除子挂载点，都会体现在另一方，即`foo <--> bar`  
- `slave`属性：类似shared，但只是单向的，foo下新增或移除子挂载点会体现在bar中，但bar中新增或移除子挂载点不会影响foo，即，即`foo --> bar, bar -x-> foo`  
- `unbindable`属性：表示挂载点bar目录将无法执行bind操作关联到其它挂载点  

### shared类型

例如，foo bind到bar时，将挂载点bar设置为shared：

```bash
sudo mount --bind foo bar
sudo mount --make-shared bar
# 或者一条命令：mount --make-shared --bind foo bar
```

现在bar挂载点是shared状态，该状态表示：在互相绑定的foo或bar下新增或移除子挂载点时，会同步体现在另一方。

例如，在foo下挂载一个新挂载点：

```bash
$ mkdir baz
$ mkdir foo/subfoo  # foo/下创建的文件会同步到bar/
$ ls foo bar
bar: subfoo
foo: subfoo
# 在foo下新增子挂载点
# foo/subfoo和bar/subfoo都将和baz关联
$ sudo mount --bind baz foo/subfoo

$ ls baz
subbaz
$ ls foo/subfoo/
subbaz
$ ls bar/subfoo/
subbaz
```

可以查看/proc/self/mountinfo文件查看某个目录是否已被挂载，以及某个挂载点是否处于shared状态：

```bash
$ grep 'foo' /proc/self/mountinfo
622 29 8:5 /home/longshuai/fs/foo /home/longshuai/fs/bar rw,relatime shared:1
683 622 8:5 /home/longshuai/fs/baz /home/longshuai/fs/bar/subfoo rw,relatime shared:1
684 29 8:5 /home/longshuai/fs/baz /home/longshuai/fs/foo/subfoo rw,relatime shared:1
```

当挂载点信息中有shared时，表明该挂载点具有shared属性，`shared:1`中的1表示这个挂载点在peer group 1中。

当两个挂载点在同一个peer group中时，该组中的任何一个挂载点下新增或移除子挂载点，都会体现在组中其他挂载点目录下。

卸载这几个挂载点，以便后续继续做实验：

```bash
$ sudo umount foo/subfoo  # 会同步卸载bar/subfoo挂载点
$ sudo umount bar
```

### private类型

例如，foo bind到bar时，将bar设置为private：

```bash
$ sudo mount --make-private --bind foo bar
$ tree foo bar
foo
└── subfoo
bar
└── subfoo

# 挂载点没有shared属性
$ grep 'foo' /proc/self/mountinfo         
622 29 8:5 /home/longshuai/fs/foo /home/longshuai/fs/bar rw,relatime
```

现在无论是在foo/subfoo还是在bar/subfoo上挂载子挂载点，都不会同步影响另一方：

```bash
$ sudo mount --bind baz foo/subfoo

$ mkdir -p bazz/subbazz
$ sudo mount --bind bazz bar/subfoo/

$ tree baz foo/subfoo bazz bar/subfoo/
baz
└── subbaz
foo/subfoo
└── subbaz
bazz
└── subbazz
bar/subfoo/
└── subbazz
```

卸载以便后续实验：

```bash
$ sudo umount bar/subfoo foo/subfoo bar
```

### slave类型

slave类型类似于shared类型，但它是单向同步。

```bash
sudo mount --make-slave --bind foo bar
```

查看`/proc/self/mountinfo`会看到`master`：

```bash
$ grep 'foo' /proc/self/mountinfo
622 29 8:5 /home/longshuai/fs/foo /home/longshuai/fs/bar rw,relatime master:1
```

现在，在foo下添加或移除子挂载点，会同步到bar挂载点，而在bar下添加或移除子挂载点，不会影响foo。

```bash
# 在foo下添加子挂载点，bar下将也有该挂载点
$ sudo mount --bind baz foo/subfoo

$ tree foo bar
foo
└── subfoo
    └── subbaz
bar
└── subfoo
    └── subbazz

# 在bar下添加子挂载点，子挂载点不会同步到foo下
$ mkdir bar/subbar
$ sudo mount --bind bazz bar/subbar

$ tree foo/subbar bar/subbar
foo/subbar
bar/subbar
└── subbazz
```

### unbindable类型

unbindable类型主要是为了避免【将父辈或祖先挂载点挂载在子孙挂载点上】时出现的大量递归挂载的问题。

例如，当前有如下挂载信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
```

如果执行：

```bash
# --rbind类似于bind，但会递归bind当前挂载点上已有的子挂载点
mount --rbind / /home/cecilia/
```

将得到如下挂载信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sdb7 on /home/cecilia/mntY
```

如果再执行：

```bash
mount --rbind / /home/henry
```

将得到如下挂载信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sdb7 on /home/cecilia/mntY
/dev/sda1 on /home/henry
/dev/sdb6 on /home/henry/mntX
/dev/sdb7 on /home/henry/mntY
/dev/sda1 on /home/henry/home/cecilia
/dev/sdb6 on /home/henry/home/cecilia/mntX
/dev/sdb7 on /home/henry/home/cecilia/mntY
```

使用`--unbindable`属性，可避免该问题：

```bash
$ sudo mount --make-unbindable --bind foo bar
```

现在foo和bar绑定了，bar作为挂载点，它将不能再bind到其他路径：

```bash
$ sudo mount --bind bar baz
mount: /home/longshuai/fs/baz: wrong fs type, bad option, bad superblock on /home/longshuai/fs/bar, missing codepage or helper program, or other error.
```

如果是`--rbind`，假如当前有如下挂载点信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
```

执行如下操作：

```
mount --rbind --make-unbindable / /home/cecilia
```

将得到如下挂载点信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sdb7 on /home/cecilia/mntY
```

再执行如下操作：

```
mount --rbind --make-unbindable / /home/henry
```

最终得到如下挂载点信息：

```
/dev/sda1 on /
/dev/sdb6 on /mntX
/dev/sdb7 on /mntY
/dev/sda1 on /home/cecilia
/dev/sdb6 on /home/cecilia/mntX
/dev/sdb7 on /home/cecilia/mntY
/dev/sda1 on /home/henry
/dev/sdb6 on /home/henry/mntX
/dev/sdb7 on /home/henry/mntY
```

## shared subtrees的默认属性

如果bind时不指定`--make-*`，它默认的shared subtrees属性是什么呢？这个问题有点复杂。

首先要理解父子挂载点对shared subtrees属性的继承规则。

- 如果父挂载点当前的属性为shared，则子挂载点的默认属性也将为shared  
- 只要父挂载点当前属性不是shared，子挂载点的属性均为private  

注意，**内核默认的shared subtrees属性为private**，所以root挂载点`/`的属性默认是private，这将使得挂载在`/`之下的所有挂载点的默认属性都是private。但是，`/`的属性设置为shared会更方便也更符合实际需求。

所以，**systemd系统在初始化内核过程中，在挂载`/`时，会将其设置为shared属性**。

```bash
$ grep '/ / ' /proc/self/mountinfo 
29 1 8:5 / / rw,relatime shared:1
```

目前主流Linux大多数都使用systemd系统，由于此时`/`被设置为shared，使得挂载在`/`之下的所有挂载点的默认属性都是shared。

```bash
$ grep -E '/ / |foo' /proc/self/mountinfo 
29 1 8:5 / / rw,relatime shared:1
622 29 8:5 foo bar rw,relatime shared:1
```

### mount namespace的shared subtrees默认属性

对于mount bind来说，默认shared subtrees属性的规则确实如上所述。

但在Linux中，**除了mount bind会应用shared subtrees属性，mount namespace也会应用shared subtrees属性**。

创建mount namespace时，会拷贝当前的挂载点信息到新的mnt namespace中，但用户创建namespace，一般更希望的是实现一个完全隔离的运行环境。所以，对于mount namespace来说，默认的shared subtrees属性都是private。即相当于在mnt namespace中的挂载点执行了：

```bash
mount --make-private /
```

这会使得`/`之下的挂载点的share subtrees属性默认都是private。

```bash
$ sudo mount --bind --make-shared foo bar  # 设置为shared
$ sudo unshare --mount --uts /bin/bash     # 创建mnt+uts namespace
$ grep -E '/ / |foo' /proc/self/mountinfo  # 默认都是private
682 681 8:5 / / rw,relatime - ext4 /dev/sda5 rw,errors=remount-ro
944 682 8:5 /home/longshuai/fs/foo /home/longshuai/fs/bar rw,relatime
```

如果想要禁止mnt namespace中shared subtrees属性默认设置为private的行为，可在创建mount namespace时加上选项`--propagation unchanged`，表示之前shared subtrees是什么属性，创建的mnt namespace中就是什么属性。

除了`unchanged`表示复制上级namespace挂载点的shared subtrees属性外，还可以直接设置所创建的mnt namespace中shared subtrees的默认属性：

```bash
unshare --mount --propagation private    # 这是默认的
unshare --mount --propagation shared
unshare --mount --propagation slave
# 没有unbindable属性，因为unbindable是针对mount bind的
```