---
title: 获取shell脚本所在目录
p: shell/get_script_dir.md
date: 2021-01-25 18:20:41
tags: Shell
categories: Shell
---

--------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**

--------

# 获取shell脚本所在目录

当要在脚本中使用该脚本的相对路径时，需要获取该脚本所在的脚本目录，这是非常常见的需求。例如，在a.sh中想要判断a.sh所在目录下是否有一个名为utils.sh的shell脚本，如果有则执行。

获取脚本所在路径，直接在脚本中使用pwd命令是不可行的，pwd获取的是当前工作目录的路径。例如在/tmp目录下执行\~/junma/a.sh时，a.sh中的pwd将输出/tmp。

也不能在脚本中使用`dirname $0`来获取该脚本的路径。因为如果脚本a.sh是在脚本c.sh中被source加载的，那么a.sh中的`$0`将是c.sh，`dirname $0`获取到的将是c.sh所在的目录。

**当要获取脚本所在路径时，无论该脚本是被直接执行，还是被source加载，都应当使用`${BASH_SOURCE[0]}`**。

如果a.sh中source b.sh，b.sh中又source c.sh，如果想要获取最顶层a.sh脚本的路径，可以在a.sh、b.sh、c.sh中使用`dirname $0`，也可以使用`dirname ${BASH_SOURCE[-1]}`。

即：

- 想要获得当前脚本的路径，使用`${BASH_SOURCE[0]}`  
- 想要获得最顶层执行source命令的主调脚本的路径，使用`${BASH_SOURCE[-1]}`或`$0`  

另外，有些脚本是软链接文件，使用dirname之前，应当先使用realpath命令获取原始文件的绝对路径。

例如，下面是进入当前脚本所在目录的一种方式：

```bash
cd "$(dirname "$(realpath "${BASH_SOURCE[0]}")")"
```

在每个需要获取脚本所在路径的脚本中都写这么一行啰嗦的代码，是一件很烦的事情。因此，可定义一个shell函数，将其放在`~/.bashrc`中。

```bash
#### function get_script_dir start ####
# Usage: display the directory of current script
#  1. if this function used in sourcing script, 
#       get_script_dir -s
#  2. if this function used in binary script,
#       get_script_dir
function get_script_dir(){
  # for i in "${BASH_SOURCE[@]}";do echo $i; done
  local sourced=0
  [ "x$1" = "x-s" ] && sourced=1
  if [ $sourced -eq 1 ];then
    dirname "$(realpath "${BASH_SOURCE[1]}")"
  else
    dirname "$(realpath "${BASH_SOURCE[-1]}")"
  fi
}

export -f get_script_dir
```

注意，上面使用`${BASH_SOURCE[-1]}`获取以二进制方式执行的shell脚本所在的路径，使用`${BASH_SOURCE[1]}`获取以source方式加载的shell脚本所在路径。

之所以使用index=1而不是index=0，是因为该函数该函数定义在.bashrc中，该文件~/.bashrc也是被source加载的，因此`${BASH_SOURCE[0]}`的值总是.bashrc。

或者也可以这样理解，任何一个被调用shell函数内部的`${FUNCNAME[0]}`都保存了该函数的名称，而`${FUNCNAME[N]}`总是定义于`${BASH_SOURCE[N]}`中，被调用于`${BASH_SOURCE[N+1]}`中。因此，在a.sh中调用.bashrc中定义的`get_script_dir`函数，该函数内部的`${FUNCNAME[0]}`是函数名，`${BASH_SOURCE[0]}`是.bashrc，`${BASH_SOURCE[1]}`是a.sh。

定义好`get_script_dir`函数之后，source加载.bashrc，然后就可以在任意shell脚本中使用该函数。例如，通过调用该函数来进入脚本所在目录：

```bash
# 在被执行的shell脚本中
cd "$(get_script_dir)"

# 在被source的shell脚本中
cd "$(get_script_dir -s)"
```

