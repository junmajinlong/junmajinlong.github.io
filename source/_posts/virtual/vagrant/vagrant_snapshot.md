---
title: 熟练使用vagrant(13)：vagrant管理快照
p: virtual/vagrant/vagrant_snapshot.md
date: 2020-10-01 17:37:42
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(13)：vagrant管理快照

vagrant支持为虚拟机拍摄快照来保存当前状态，以后可以方便地恢复到指定的快照。

```powershell
$ vagrant snapshot -h
Usage: vagrant snapshot <subcommand> [<args>]

Available subcommands:
     delete      # 删除快照
     list        # 列出已有的快照
     pop         # 恢复到最近一个快照并删除该快照
     push        # 创建一个快照
     restore     # 恢复到指定的快照
     save        # 创建一个快照，并指定快照名称
```

save/restore和push/pop是类似的，其实后者是前者的一种简单操作。前者要指定名称，后者自动分配名称。但官方手册建议，不要混用它们，push和pop一起使用，save和restore一起使用。

例如：
```powershell
# 为虚拟机vm1创建一个快照，快照名为vm1_snapshot1
vagrant snapshot save vm1 vm1_snapshot1

# 使用push创建一个快照，名称自动分配
vagrant snapshot push vm1  # 分配的快照名push_1602923748_6765

# 使用pop恢复到最近一个快照，默认会同时删除该快照
# 使用--no-start可指定恢复快照后不启动虚拟机
# 使用--no-delete可指定pop恢复快照时不删除该快照
vagrant snapshot pop vm1
vagrant snapshot pop vm1 --no-start
vagrant snapshot pop vm1 --no-delete

# 使用restore恢复到指定快照，不会删除任何快照
vagrant snapshot restore vm1 vm1_snapshot1

# 恢复快照后，不要启动虚拟机
vagrant snapshot restore vm1 vm1_snapshot1 --no-start
```
