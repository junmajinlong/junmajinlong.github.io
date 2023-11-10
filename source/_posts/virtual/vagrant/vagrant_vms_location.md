---
title: 熟练使用vagrant(5)：vagrant的虚拟机安装在哪里
p: virtual/vagrant/vagrant_vms_location.md
date: 2020-10-01 17:37:34
tags: Virtualization
categories: Virtualization
---

--------

- **目录：[熟练使用vagrant系列文章](/virtual/index#vagrant)**  
- **vagrant视频教程**：[熟练使用vagrant管理虚拟机](https://edu.51cto.com/sd/304f8)

--------


# 熟练使用vagrant(5)：vagrant的虚拟机安装在哪里

在`vagrant up`初始化并启动虚拟机后，在Vagrantfile文件所在目录内，将有一个名为`.vagrant`的目录，`vagrant up`根据Vagrantfile所创建的虚拟机的vagrant状态数据都处于`.vagrant/machines`内。

虚拟机自身安装到了哪里？这和虚拟机管理工具有关：
- 对于hyperv来说，虚拟机自身也被安装在`.vagrant/machines`中  
- 对于virtualbox来说，虚拟机自身则被安装在virtualbox所指定的默认安装目录下  

```
# 对于hyperv，虚拟机直接安装在.vagrant/machines内
$ tree -a
.
├── .vagrant
│   ├── machines
│   │   └── default
│   │       └── hyperv
│   │           ├── Snapshots
│   │           ├── Virtual Hard Disks
│   │           │   └── generic-ubuntu2004-hyperv.vhdx   # 磁盘文件
│   │           ├── Virtual Machines
│   │           │   ├── F80DC6C1-39DF-444A-8589-49574F8E378F
│   │           │   ├── F80DC6C1-39DF-444A-8589-49574F8E378F.VMRS
│   │           │   ├── F80DC6C1-39DF-444A-8589-49574F8E378F.vmcx
│   │           │   └── F80DC6C1-39DF-444A-8589-49574F8E378F.vmgs
│   │           ├── action_configure
│   │           ├── action_provision
│   │           ├── action_set_name
│   │           ├── box_meta
│   │           ├── creator_uid
│   │           ├── id
│   │           ├── index_uuid
│   │           ├── private_key
│   │           ├── synced_folders
│   │           └── vagrant_cwd
│   └── rgloader
│       └── loader.rb
└── Vagrantfile

# 对于virtualbox，只保存了部分状态文件
$ tree -a
.
├── .vagrant
│   ├── machines
│   │   └── ubuntu2004_junmajinlong
│   │       └── virtualbox
│   │           ├── action_provision
│   │           ├── action_set_name
│   │           ├── box_meta
│   │           ├── creator_uid
│   │           ├── id
│   │           ├── index_uuid
│   │           ├── private_key
│   │           ├── synced_folders
│   │           └── vagrant_cwd
│   └── rgloader
│       └── loader.rb
└── Vagrantfile

# 实际的虚拟机保存在virtualbox的默认安装目录vbox_xuniji内
$ tree vbox_xuniji/ubuntu2004/
vbox_xuniji/ubuntu2004/
├── Logs
│   ├── VBox.log
│   ├── VBoxHardening.log
│   └── VBoxUI.log
├── generic-ubuntu2004-virtualbox-disk001.vmdk
├── ubuntu2004.vbox
└── ubuntu2004.vbox-prev
```
