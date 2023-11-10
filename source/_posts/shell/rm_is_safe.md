---
title: 更安全的rm命令，保护重要数据
p: shell/rm_is_safe.md
date: 2020-05-05 18:20:43
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 更安全的rm命令，保护重要数据

网上流传的安全的rm，几乎都是提供一个rm的"垃圾"回收站，在服务器环境上来说，这实非良方。

我想，提供一个安全的rm去保护一些重要的文件或目录不被删除，避免出现重要数据误删的悲剧，或许才是更佳方案。

我写了一个脚本：https://github.com/malongshuai/rm_is_safe，源码和用法本文后面已经提供了。

![](/img/stuffs/a13.gif)
## 工作方式

`rm_is_safe`会创建一个名为`/bin/rm`的shell脚本，同时会备份原生的/bin/rm为/bin/rm.bak。所以，原来如何使用rm，现在也以一样的方式使用rm，没有任何区别。

为了区分原生rm和伪装后的安全的rm，下面将伪装的rm命令称为`rm_is_safe`。

`rm_is_safe`会自动检查rm被调用时传递的参数，如果参数中包含了重要文件，可能意味着这是一次危险的rm操作，`rm_is_safe`会直接忽略本次rm。至于哪些属于重要文件，由你自己来决定。

`rm_is_safe`对所有用户都有效，包括目前已存在的用户和未来新创建的用户。

## 哪些是重要文件？

1. 根目录`/`以及根目录下的子目录、子文件总是自动被保护的  
2. 你可以在`/etc/security/rm_fileignore`中定义你自己觉得重要的文件，每行定义一个被保护的文件路径。例如：

    ```
    /home/junmajinlong
    /home/junmajinlong/apps
    ```

现在，该文件中定义的两个文件都被保护起来了，它们是安全的，不会被rm删除。

**注意事项:**  

1. 显然，被保护的目录是不会进行递归的，所以'/bin'是安全的，而'/bin/aaa'是不安全的，除非你将它加入/etc/security/rm_fileignore文件中  
2. 根目录`/`以及根目录下的子目录是自动被保护的，不用手动将它们添加到/etc/security/rm_fileignore中  
3. /etc/security/rm_fileignore文件中定义的路径可以包含任意斜线，`rm_is_safe`会自动处理。所以，'/home/junmajinlong'和'/home///junmajinlong/////'都是有效路径  
4. /etc/security/rm_fileignore中定义的路径中不要使用通配符，例如`/home/*`是无效的  
5. 不要在/etc/security/rm_fileignore使用相对路径  

## Usage

1.执行后文提供的Shell脚本：

```
$ sudo bash rm_is_safe.sh
```

执行完成后，你的rm命令就变成了安全的rm了。

2.如果确实想要删除被保护的文件，比如你明确知道/data是可以删除的，那么你可以使用原生的rm命令，即/bin/rm.bak来删除。

```
$ rm.bak /path/to/file
```

3.如果你想要卸载`rm_is_safe`，执行函数`uninstall_rm_is_safe`即可：

```
# 如果找不到该函数，则先exec bash，再执行即可
$ uninstall_rm_is_safe
```

卸载完成后，`/bin/rm`就变回原生的rm命令了。


## 脚本：rm_is_safe.sh

脚本如下，假设其文件名为`rm_is_safe.sh`：

```bash
#!/bin/bash

###############################
# Author: www.junmajinlong.com
###############################

# generate /bin/rm
#   1.create file: /etc/security/rm_fileignore
#   2.backup /bin/rm to /bin/rm.bak
function rm_is_safe(){
  [ -f /etc/security/rm_fileignore ] || touch /etc/security/rm_fileignore
  if [ ! -f /bin/rm.bak ];then
    file /bin/rm | grep -q ELF && /bin/cp -f /bin/rm /bin/rm.bak
  fi

  cat >/bin/rm<<'eof'
#!/bin/bash
args=$(echo "$*" | tr -s '/' | tr -d "\042\047" )
safe_files=$(find / -maxdepth 1 | tr '\n' '|')$(cat /etc/security/rm_fileignore | tr '\n' '|')
echo "$args" | grep -qP "(?:${safe_files%|})(?:/?(?=\s|$))"
if [ $? -eq 0 ];then
  echo -e "'\e[1;5;33mrm $args\e[0m' is not allowed,Exit..."
  exit 1
fi
/bin/rm.bak "$@"
eof

  chmod +x /bin/rm
}

# for uninstall rm_is_safe
# function `uninstall_rm_safe` used for uninstall
function un_rm(){
  # make efforts for all user
  if [ ! -f /etc/profile.d/rm_is_safe.sh ];then
    shopt -s nullglob
    for uh in /home/* /root /etc/skel;do
      shopt -u nullglob

cat >>$uh/.bashrc<<'eof'
# for rm_is_safe:
[ -f /etc/profile.d/rm_is_safe.sh ] && source /etc/profile.d/rm_is_safe.sh
eof
    done
  fi

cat >/etc/profile.d/rm_is_safe.sh<<'eof'
function uninstall_rm_is_safe(){
  unset uninstall_rm_is_safe
  /bin/unlink /etc/security/rm_fileignore
  /bin/cp -f /bin/rm.bak /bin/rm
  /bin/unlink /etc/profile.d/rm_is_safe.sh
  shopt -s nullglob
  for uh in /home/* /root /etc/skel;do
    shopt -u nullglob
    sed -ri '\%# for rm_is_safe%,\%/etc/profile.d/rm_is_safe.sh%d' $uh/.bashrc
  done
}
export -f uninstall_rm_is_safe
eof
}

rm_is_safe
un_rm
```


