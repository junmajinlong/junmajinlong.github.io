---
title: 16.成就感源于创造：自己动手写Ansible模块
p: ansible/16_write_ansible_module.md
date: 2021-07-11 10:37:45
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 16.成就感源于创造：自己动手写Ansible模块

从小，书上就告诉我们"自己动手，丰衣足食"，但是在IT领域里，这句话不完全对，其实自己不动手，也可以丰衣足食，因为这个领域里提倡的是"不要重复造轮子"，别人已经把轮子造好了，直接拿来用就好，简单又高效。但自己造轮子，总是有好处的，至少收获了造轮子的过程，有些轮子，是前进道路上必造不可的。

闲话只扯一段。Ansible提供了大量已经造好的轮子，几千个模块(此刻是3387个)、很多个插件可以供用户直接使用，基本上能解决绝大多数场景的需求。但是，再多的模块也不够用，总有一些需求是只属于自己的。这时，就需要造一个只适合自己想法的轮子，根据自己的需求去扩展Ansible的功能。

Ansible允许用户自定义的方式扩展很多方面的功能，包括：  
- (1).实现某功能的模块，模块以作为任务的方式被执行  
- (2).各类插件，用于调整Ansible的行为。目前Ansible有12类插件  

```shell
$ grep 'plugins/' /etc/ansible/ansible.cfg 
#action_plugins     = /usr/share/ansible/plugins/action
#become_plugins     = /usr/share/ansible/plugins/become
#cache_plugins      = /usr/share/ansible/plugins/cache
#callback_plugins   = /usr/share/ansible/plugins/callback
#connection_plugins = /usr/share/ansible/plugins/connection
#lookup_plugins     = /usr/share/ansible/plugins/lookup
#inventory_plugins  = /usr/share/ansible/plugins/inventory
#vars_plugins       = /usr/share/ansible/plugins/vars
#filter_plugins     = /usr/share/ansible/plugins/filter
#test_plugins       = /usr/share/ansible/plugins/test
#terminal_plugins   = /usr/share/ansible/plugins/terminal
#strategy_plugins   = /usr/share/ansible/plugins/strategy
```

本文会以最简单易懂的方式介绍模块的自定义方式(不介绍自定义插件，内容太多)。考虑到有些人没有编程基础，这里我将同时使用Shell和Python来介绍，看菜吃饭即可。

## 16.1 自定义模块简介

可以使用任何一种语言来自定义模块，只要目标主机能执行即可。这得益于Ansible的运行方式：将模块的代码发送到目标节点上调用对应的解释器执行。

比如可以使用Shell脚本自定义模块，Ansible执行时会将Shell脚本的内容发送到目标节点上并调用目标节点的shell解释器(比如bash)去执行这些命令行。

但是绝大多数的模块都是使用Python语言编写的(Windows相关模块除外，它们使用PowerShell编写)。如果自己定义模块，按理说也建议使用Python语言，Ansible为自定义模块提供了不少非常方便的接口。但是，如果熟悉Shell脚本的话，对于逻辑不太复杂的功能，Shell脚本比Python要方便简洁，所以不要觉得用Shell写Ansible模块上不了台面。

### 16.1.1 自定义模块前须知：模块的存放路径

当编写好自己的模块后，需要让Ansible知道在哪里可以找到它。有三个位置可以存放模块：  

- (1).playbook文件所在目录的library目录内，即pb.yml/../library目录内  
- (2).roles/Role_Name/library目录内  
- (3).ansible.cfg中library指令指定的目录或者环境变量ANSIBLE_LIBRARY指定的目录  

显然，`roles/Role_Name/library`目录内的模块只对该Role有效，而`pb.yml/../library`对所有Role和pb.yml有效，ansible.cfg中library指令指定的路径对全局有效。

### 16.1.2 自定义模块前须知：模块的要求

从第一章节到现在，已经学习了很多模块，相信对模块的使用已经非常熟悉。我想各位在使用各个模块的过程中，应该能感受到这些模块的一些共同点：  

- (1).每个模块都有changed、failed状态  
- (2).绝大多数模块都可以提供各种各样的选项参数  
- (3).很多模块都要求必须有某些选项参数  
- (4).每个模块都有返回值，从而可以通过register注册成变量  
- (5).返回值全都是json格式  
- (6).有些模块具有幂等性  
- (7)....

![](/img/ansible/1625971915233.png)

所以，这跟写一个脚本或写一个程序没什么区别，仅仅只是在写这些脚本时多了一些特殊要求。

下面将先以Shell脚本的方式解释并定义模块，稍后再演示一个简单的Python定义的模块，主要是为了体现Ansible为Python自定义模块所提供的方便的功能。

## 16.2 Shell脚本自定义模块(一)：Hello World

先使用Shell脚本一个最简单的自定义模块，只显示"hello world"。

下面是脚本模块的内容，其路径为library/say_hello.sh：

```shell
#!/bin/bash

echo '{"changed": false, "msg": "Hello World"}'
```

注意上面的false不要加引号，因为它要表示的是json的布尔假，加上引号就成了json中的字符串类型。

在library同级目录下创建一个playbook文件shell_module1.yml，在其中使用该模块：

```yaml
---
- hosts: localhost
  gather_facts: no
  tasks: 
    - name: say hello with my module
      say_hello:
      register: res
    
    - debug: var=res
    - debug: var=res.msg
```

执行：

```shell
$ ansible-playbook shell_module1.yml     

PLAY [localhost] *****************************

TASK [say hello with my module] **************
ok: [localhost]

TASK [debug] *********************************
ok: [localhost] => {
    "res": {
        "changed": false,
        "failed": false,
        "msg": "Hello World"
    }
}

TASK [debug] **********************************
ok: [localhost] => {
    "res.msg": "Hello World"
}

PLAY RECAP ************************************
localhost  : ok=3    changed=0    unreachable=0
```

是否发现自定义模块好简单呢？

## 16.3 Shell脚本自定义模块(二)：简版file模块

Ansible的file模块可以创建、删除文件或目录，这里通过Shell脚本的方式来自定义精简版的file模块，称为`file_by_shell`。

这个模块能够处理相关参数，包括：  
- (1).识别路径和文件名，选项名称假设为path，该选项必须不能省略  
- (2).识别是创建还是删除操作，选项名称假设为state，它只能是两种值：present或absent，默认是present  
- (3).识别在创建操作时，创建的是文件还是目录(此处不考虑其它文件类型)，如果路径不存在，这里决定递归创建缺失的目录，该选项名称为type，该选项在创建操作时必须不能省略，它只能有两种值：file或directory  
- (4).识别在创建操作时，是否给定了权限参数，比如指定要创建的文件权限为0644，选项名称为mode  
- (5).识别在创建操作时，是否给定了owner、group  

其它的功能此处不多考虑。

所以，按照常规的脚本调用方式，这个shell脚本的用法大概如下：

![](/img/ansible/1625971861808.png)

例如，Ansible会先把参数以如下方式写进临时文件xxx.tmp：

```
path=PATH state=STATE type=TYPE...
```

然后将这个临时文件名传递给模块文件：

```shell
file_by_shell.sh xxx.tmp
```

所以，Shell脚本必须得能够从这个临时文件中处理来自playbook中的选项和参数。比如可以像下面这样获取path选项的值。

```shell
cat xxx.tmp | tr ' ' '\n' | sed -nr 's/^path=(.*)/\1/'
```

有了这些背景知识，再来写Shell脚本`file_by_shell.sh`：

```shell
#!/bin/bash

############# 定义一个输出信息到标准错误的函数 #############
function err_echo(){ echo "$@" >&2;exit 1; }

############# 选项、参数解析 #############
args="$(cat "$@" | tr ' ' '\n')"
path="$(echo "$args"  | sed -nr 's/^path=(.*)/\1/p')"
type="$(echo "$args"  | sed -nr 's/^type=(.*)/\1/p')"
state="$(echo "$args" | sed -nr 's/^state=(.*)/\1/p')"
mode="$(echo "$args"  | sed -nr 's/^mode=(.*)/\1/p')"
owner="$(echo "$args" | sed -nr 's/^owner=(.*)/\1/p')"
group="$(echo "$args" | sed -nr 's/^group=(.*)/\1/p')"

############# 处理选项之间的逻辑 #############

# path选项：必须存在
[ "$path" ] || { err_echo "'path' argument missing"; }

# state选项：如果不存在，则默认为present
# 且state只能为两种值：present或absent
: ${state="present"}
[ "$(echo $state | sed -r 's/(present|absent)//')" ] && { 
    err_echo "'state' argument error";
}

# type选项：在创建操作时必须存在，且只能为两种值：file或directory
if [ "${state}x" == "presentx" ];then
  [ "${type}" ] || { err_echo "'type' argument missing"; }
  
  # type的值只能是：file或directory
  [ "${type}" != "file" ] && [ "${type}" != "directory" ] && { 
    err_echo "'type' argument error";
  }
fi

# mode选项：如果该选项存在，必须是3位或4位数，且每位都是小于7的数值
# 如果该选项不存在，则不管，即按照对应用户的umask值决定权限
if [ "$mode" ];then
  echo $mode | grep -E '[0-7]?[0-7]{3}' &>/dev/null || {
    err_echo "'mode' argument error";
  }
fi

############# 定义函数，输出json格式并退出 #############
function echo_json() {
  echo '{ ' $1 ' }'
  # 输出完后正常退出
  exit
}

############# 删除文件/目录的逻辑 #############

# 为了实现幂等性，先判断目标是否存在，
# 如果存在，不管是文件还是目录，都删除，并设置changed=true
# 如果不存在，什么也不做，并设置changed=false
# 如果报错(比如权限不足)，无视，Ansible会自动获取报错信息
if [ $state == "absent" ];then
  if [ -e "$path" ];then
    rm -rf "$path"
    return_str='"changed": true'
    echo_json "$return_str"
  else
    return_str='"changed": false'
    echo_json "$return_str"
  fi
fi

############# 创建文件/目录的逻辑 #############

# 为了实现幂等性，先判断目标是否存在，
# 如果存在，且类型匹配，则什么也不做，并设置changed=false
# 如果存在，但类型不匹配(比如想要创建文件，但已存在同名目录)，则报错  
# 如果不存在，则根据类型创建文件/目录
# 如果报错(如权限不足)，无视，Ansible会自动获取报错信息
if [ $state == "present" ];then
  if [ -e "$path" ];then
    # 文件已存在

    # 获取已存在文件的类型
    file_type=$(ls -ld "$path" | head -c 1)
    if [ $file_type == "-" -a "$type" == "file" ] || \
       [ $file_type == "d" -a "$type" == "directory" ]
    then
      # 类型匹配
      return_str='"changed": false'
      echo_json "$return_str"
    else
      # 类型不匹配
      err_echo "target exists but filetype error";
    fi
  else
    # 文件/目录不存在，在此处创建，同时创建缺失的上级目录
    dir="$(dirname "$path")"
    [ -d "$dir" ] || mkdir -p "$dir"
    [ $type = "file" ] && touch "$path"
    [ $type = "directory" ] && mkdir "$path"

    # 设置权限、owner、group，如果属性修改失败，则删除已创建的目标
    [ "$mode" ]  && { chmod $mode  "$path" || rm -rf "$path"; }
    [ "$owner" ] && { chown $owner "$path" || rm -rf "$path"; }
    [ "$group" ] && { chgrp $group "$path" || rm -rf "$path"; }

    return_str='"changed": true'
    echo_json "$return_str"
  fi
fi
```

再写一个playbook文件shell_module.yml来使用该shell脚本模块：

```yaml
---
- hosts: localhost
  gather_facts: no
  tasks: 
    - name: use file_by_shell module to create file
      file_by_shell: 
        # 注:父目录test不存在
        path: /tmp/test/file1.txt
        state: present
        type: file
        mode: 655
    - name: use file_by_shell module to create directory
      file_by_shell: 
        path: /tmp/test/dir1
        state: present
        type: directory
        mode: 755
```

执行该playbook：

```shell
$ ansible-playbook shell_module.yml

PLAY [localhost] *************************************

TASK [use file_by_shell module to create file] *******
changed: [localhost]

TASK [use file_by_shell module to create directory] **
changed: [localhost]

PLAY RECAP *******************************************
localhost     : ok=2    changed=2    unreachable=0
```

再次执行，因具备幂等性，所以不会做任何操作：

```shell
$ ansible-playbook shell_module.yml

PLAY [localhost] ******************************************

TASK [use file_by_shell module to create file] ************
ok: [localhost]

TASK [use file_by_shell module to create directory] *******
ok: [localhost]

PLAY RECAP ************************************************
localhost           : ok=2    changed=0
```

执行删除操作：

```yaml
---
- hosts: localhost
  gather_facts: no
  tasks: 
    - name: use file_by_shell module to remove file
      file_by_shell: 
        path: /tmp/test/file1.txt
        state: absent
    - name: use file_by_shell module to remove directory
      file_by_shell: 
        path: /tmp/test/dir1
        state: absent
```

执行：

```
PLAY [localhost] ****************************************

TASK [use file_by_shell module to remove file] **********
changed: [localhost]

TASK [use file_by_shell module to remove directory] *****
changed: [localhost]

PLAY RECAP **********************************************
localhost                  : ok=4    changed=2
```

## 16.4 Python自定义模块

使用Python定义模块，要简单一些，一方面是因为Python的逻辑处理能力比Shell要强，另一方面是因为Ansible提供了一些方便的Python接口，比如处理参数、处理退出时返回的json数据。

这里仍然使用Python编写一个自定义的精简版的file模块。

对于初学者来说，使用Python自定义模块的第一步，是先搭建好脚本的框架：

```python
#!/usr/bin/python

from ansible.module_utils.basic import *

def main():
  ...to_do...

if __name__ == '__main__':
  main()
```

所有的处理逻辑都在main()函数中定义。

下一步是构造一个模块对象，并处理Ansible传递给脚本的参数。

```python
#!/usr/bin/python

from ansible.module_utils.basic import *

def main():
  md = AnsibleModule(
    argument_spec = dict(
      path  = dict(required=True, type='str'),
      state = dict(choices=['present', 'absent'], type='str', default="present"),
      type  = dict(type='str'),
      mode  = dict(type='int',default=None),
      owner = dict(type='str',default=None),
      group = dict(type='str',default=None),
    )
  )
  params = md.params
  path = params['path']
  state = params['state']
  target_type = params['type']
  # 权限位读取时是十进制，要将其看作是8进制值而不能是十进制
  mode = params['mode'] and int(str(params['mode']),8)
  owner = params['owner']
  group = params['group']

if __name__ == '__main__':
  main()
```

上面使用AnsibleModule()构造了一个Ansible模块对象md，并指定处理的参数以及相关要求。比如要求path必须存在，且其数类型是str，比如state默认值为present，且只有两种值可选。关于参数处理，完整的用法参考官方手册：<https://docs.ansible.com/ansible/latest/dev_guide/developing_program_flow_modules.html#ansiblemodule>。

需要注意的是，无法直接在此对各选项之间的逻辑关系进行判断，例如创建目标时必须指定文件类型。所以，这种逻辑要么单独对选项关系做判断，要么在执行操作(比如创建文件)时在对应函数中进行判断或异常处理。

为图简单，我直接在获取完参数之后立即对它们做判断：(为节省篇幅，我省略一部分已编写的代码)

```python
#!/usr/bin/python

from ansible.module_utils.basic import *

def main():
  ......
  path = params['owner']
  group = params['group']

  # present时，必须指定type参数
  if state == 'present' and target_type is None:
    raise Exception('type argument missing')
  
if __name__ == '__main__':
  main()
```

在之后，是定义删除、创建以及退出时的逻辑：

```python
#!/usr/bin/python

from ansible.module_utils.basic import *

def main():
  ......
  # present时，必须指定type参数
  if state == 'present' and target_type is None:
    raise AnsibleModuleError(results={'msg': 'type argument missing'})
  
  # 如果是absent，直接删除
  # 否则是创建目标，需要区分要创建的目标类型
  if state == 'absent':
    result = remove_target(path)
  elif target_type == 'file':
    result = create_file(path, mode, owner, group)
  elif target_type == 'directory':
    result = create_dir(path, mode, owner, group)
  
  # 退出，并输出json数据
  md.exit_json(**result)
  
if __name__ == '__main__':
  main()
```

最后，就是定义这三个函数：`remove_target、create_file、create_dir`。因为这三个函数中都要判断目标是否存在以及文件类型，所以将判断是否存在的逻辑也定义成一个函数`get_stat`。

下面是`get_stat`的代码：

```python
# 这里用到了os，所以记得导入os
# 这里用到了to_bytes，记得导入
# import os
# from ansible.module_utils._text import to_bytes

def get_stat(path):
  b_path = to_bytes(path)
  # 目标存在
  try: 
    if os.path.lexists(b_path):
      if os.path.isdir(b_path):
        return 'directory'
      else:
        return 'file'  # 不管其它类型，此处认为除了目录，就是文件
    # 目标不存在
    else:
      return False
  except OSError: 
    raise
```

下面是`remove_target`代码：

```python
# 这里使用了shutil，所以记得导入shutil
# import shutil

def remove_target(path):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  try:
    # 存在时，删除
    if target_stat:
      if target_stat == 'directory':
        shutil.rmtree(b_path)
      else:
        # 删除目录
        os.unlink(b_path)

      result.update({'changed': True})
    # 不存在时，不管
    else:
      result.update({'changed': False})
  except Exception: 
    raise
  return result
```

下面是创建普通文件的函数`create_file`代码：

```python
def create_file(path, mode=None, owner=None, group=None):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  # 目标存在，则判断已存在的类型
  # 目标不存在，则创建文件
  if target_stat:
    # 如果已存在的不是普通文件类型，报错
    if target_stat != 'file':
      raise Exception('target already exists, but type error')
    result.update({'changed': False})
  else:
    # 目标不存在，创建文件
    try:
      # 父目录是否存在？不存在则递归创建
      if not get_stat(os.path.dirname(b_path)):
        os.makedirs(os.path.dirname(b_path))
      open(b_path, 'wb').close()
    except (OSError, IOError):
      raise

    # 决定是否更改权限、owner、group，如果更改出错，删除已创建的文件
    try:
      if mode:
        os.chmod(b_path, mode)
      if owner:
        os.chown(b_path, pwd.getpwnam(owner).pw_uid, -1)
      if group:
        os.chown(b_path, -1, pwd.getpwnam(group).pw_gid)
      result.update({'changed': True})
    except Exception:
      if not target_stat:
        os.remove(b_path)
  return result
```

下面是创建普通目录的函数`create_dir`代码：

```python
def create_dir(path, mode=None, owner=None, group=None):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  # 目标存在，则判断已存在的类型
  # 目标不存在，则创建目录
  if target_stat:
    # 如果已存在的不是目录，报错
    if target_stat != 'directory':
      raise Exception('target already exists, but type error')
    result.update({'changed': False})
  else:
    # 目标不存在，创建目录
    try:
      os.makedirs(b_path)
    except (OSError, IOError):
      raise 

    # 决定是否更改权限、owner、group，如果更改出错，删除已创建的文件
    try:
      if mode:
        os.chmod(b_path, mode)
      if owner:
        os.chown(b_path, pwd.getpwnam(owner).pw_uid, -1)
      if group:
        os.chown(b_path, -1, pwd.getpwnam(group).pw_gid)
      result.update({'changed': True})
    except Exception:
      if not target_stat:
        shutil.rmtree(b_path)
  return result
```

将上面所有代码汇总，得到如下代码，并保存到文件`library/file_by_python.py`中：

```python
#!/usr/bin/python

import os
import shutil
from ansible.module_utils.basic import *
from ansible.module_utils._text import to_bytes

def get_stat(path):
  b_path = to_bytes(path)
  # 目标存在
  try: 
    if os.path.lexists(b_path):
      if os.path.isdir(b_path):
        return 'directory'
      else:
        return 'file'  # 不管其它类型，此处认为除了目录，就是文件
    # 目标不存在
    else:
      return False
  except OSError: 
    raise

def remove_target(path):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  try:
    # 存在时，删除
    if target_stat:
      if target_stat == 'directory':
        shutil.rmtree(b_path)
      else:
        # 删除目录
        os.unlink(b_path)
      
      result.update({'changed': True})
    # 不存在时，不管
    else:
      result.update({'changed': False})
  except Exception: 
    raise
  return result

def create_file(path, mode=None, owner=None, group=None):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  # 目标存在，则判断已存在的类型
  # 目标不存在，则创建文件
  if target_stat:
    # 如果已存在的不是普通文件类型，报错
    if target_stat != 'file':
      raise Exception('target already exists, but type error')
    result.update({'changed': False})
  else:
    # 目标不存在，创建文件
    try:
      # 父目录是否存在？不存在则递归创建
      if not get_stat(os.path.dirname(b_path)):
        os.makedirs(os.path.dirname(b_path))
      open(b_path, 'wb').close()
    except (OSError, IOError):
      raise

    # 决定是否更改权限、owner、group，如果更改出错，删除已创建的文件
    try:
      if mode:
        os.chmod(b_path, mode)
      if owner:
        os.chown(b_path, pwd.getpwnam(owner).pw_uid, -1)
      if group:
        os.chown(b_path, -1, pwd.getpwnam(group).pw_gid)
      result.update({'changed': True})
    except Exception:
      if not target_stat:
        os.remove(b_path)
  return result

def create_dir(path, mode=None, owner=None, group=None):
  b_path = to_bytes(path)
  target_stat = get_stat(b_path)
  result = {'path': b_path, 'target_stat': target_stat}
  
  # 目标存在，则判断已存在的类型
  # 目标不存在，则创建目录
  if target_stat:
    # 如果已存在的不是目录，报错
    if target_stat != 'directory':
      raise Exception('target already exists, but type error')
    result.update({'changed': False})
  else:
    # 目标不存在，创建目录
    try:
      os.makedirs(b_path)
    except (OSError, IOError):
      raise 

    # 决定是否更改权限、owner、group，如果更改出错，删除已创建的文件
    try:
      if mode:
        os.chmod(b_path, mode)
      if owner:
        os.chown(b_path, pwd.getpwnam(owner).pw_uid, -1)
      if group:
        os.chown(b_path, -1, pwd.getpwnam(group).pw_gid)
      result.update({'changed': True})
    except Exception:
      if not target_stat:
        shutil.rmtree(b_path)
  return result

def main():
  md = AnsibleModule(
    argument_spec = dict(
      path  = dict(required=True, type='str'),
      state = dict(choices=['present', 'absent'], type='str', default="present"),
      type  = dict(type='str'),
      mode  = dict(type='int',default=None),
      owner = dict(type='str',default=None),
      group = dict(type='str',default=None),
    )
  )
  params = md.params
  path = params['path']
  state = params['state']
  target_type = params['type']
  # 权限位读取时是十进制，要将其看作是8进制值而不能是十进制
  mode = params['mode'] and int(str(params['mode']),8)
  owner = params['owner']
  group = params['group']

  # present时，必须指定type参数
  if state == 'present' and target_type is None:
    raise Exception('type argument missing')

  # 如果是absent，直接删除
  # 否则是创建目标，需要区分要创建的目标类型
  if state == 'absent':
    result = remove_target(path)
  elif target_type == 'file':
    result = create_file(path, mode, owner, group)
  elif target_type == 'directory':
    result = create_dir(path, mode, owner, group)
  
  # 退出，并输出json数据
  md.exit_json(**result)
if __name__ == '__main__':
  main()
```

提供一个playbook文件`python_module.yml`：

```yaml
---
- hosts: localhost
  gather_facts: no
  tags: create
  tasks:
    - name: use file_by_python module to create file
      file_by_python:
        path: /tmp/test/file1.txt
        state: present
        type: file
        mode: 655
    - name: use file_by_python module to create directory
      file_by_python:
        path: /tmp/test/dir1
        state: present
        type: directory
        mode: 755

- hosts: localhost
  gather_facts: no
  tags: remove
  tasks: 
    - name: use file_by_python module to remove file
      file_by_python: 
        path: /tmp/test/file1.txt
        state: absent
    - name: use file_by_python module to remove directory
      file_by_python: 
        path: /tmp/test/dir1
        state: absent
```

执行创建操作：

```shell
$ ansible-playbook --tags create python_module.yml

PLAY [localhost] **************************************

TASK [use file_by_python module to create file] *******
changed: [localhost]

TASK [use file_by_python module to create directory] **
changed: [localhost]

PLAY [localhost] **************************************

PLAY RECAP ********************************************
localhost     : ok=2    changed=2    unreachable=0 
```

执行删除操作：

```shell
$ ansible-playbook --tags remove python_module.yml

PLAY [localhost] ***************************************

PLAY [localhost] ***************************************

TASK [use file_by_python module to remove file] ********
changed: [localhost]

TASK [use file_by_python module to remove directory] ***
changed: [localhost]

PLAY RECAP *********************************************
localhost                  : ok=2    changed=2 
```

从结果看，测试是可以通过的。

从上面分别通过Shell脚本和通过Python脚本实现精简file模块的两个示例中可以看到，完全相同的逻辑和功能，Shell脚本定义Ansible模块时不输于Python。当然，如果逻辑复杂或者遇到了Shell不好处理的逻辑，使用Python当然要优于Shell。
