---
title: 9.如虎添翼的力量：解锁强大的Jinja2模板
p: ansible/9_power_of_jinja2.md
date: 2021-07-11 10:37:38
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 9.如虎添翼的力量：解锁强大的Jinja2模板

在前面的文章中提到过很多次Jinja2，它是Python的一种模板引擎。

尽管在编写playbook时可以不用在意是否要用Jinja2，但Ansible的运行离不开Jinja2，当Ansible开始执行playbook或任务时，总是会先使用Jinja2去解析所有指令的值，然后再执行任务。另一方面，在编写任务的过程中也会经常用到Jinja2来实现一些需求。所以，Jinja2可以重要到成为Ansible的命脉。本章将学习Jinja2在Ansible中的用法。

## 9.1 Jinja2是什么？模板是什么？

何为模板？举个例子就知道了。

假设要发送一个文件给一个或多个目标节点，要发送的文件内容如下：
```
hello, __NAME__
```

其中`__NAME__`部分想要根据目标节点的主机名来确定，比如发送给www节点时内容应该为`hello, www`，发送给wwww节点时，内容应该为`hello, wwww`。换句话说，`__NAME__`是一个能够根据不同场景动态生成不同字符串的代码小片段。而根据特殊的代码片段动态生成字符串便是模板要实现的功能。

![](/img/ansible/1625971044700.png)

例如上面示例中，`__NAME__`使用了前后两个下划线包围，表示这部分是模板表达式，它是需要进行替换的，而"hello"是普通字符串，模板引擎不会去管它。

Jinja2模板引擎提供了三种特殊符号来包围模板表达式：  
- (1).{% raw %}{{xxx}}{% endraw %}：双大括号包围变量或表达式(Ansible中的变量就是它包围的)  
- (2).{% raw %}{#xxx#}{% endraw %}：Jinja2的注释符号  
- (3).{% raw %}{%xxx%}{% endraw %}`：Jinja2的一些特殊关键字标签，比如if语句、for循环语句等等  

> 注：其实还有第四种特殊符号：以#开头的行可以替代{% raw %} {%%} {% endraw %}，但一般不用。

模板更多用在web编程中来生成HTML页面，但绝不限于web编程，它可以用在很多方面，比如Ansible就使用Jinja2模板引擎来解析YAML中的字符串，也用在template模块渲染模板文件。

Jinja2的内容较多，但对于学习Ansible来说，只需要学习其中和template相关的一部分(其它的都和开发有关或Ansible中用不上)以及Ansible对Jinja2的扩展功能即可。

- (1).Jinja2的官方手册：<https://jinja.palletsprojects.com/en/2.10.x/templates/>  
- (2).Ansible Jinja2的官方手册：<https://docs.ansible.com/ansible/latest/user_guide/playbooks_templating.html>  

本章会介绍和演示其中绝大多数内容，且几乎都来自于Jinja2和Ansible的官方手册整理，足够各位的日常使用。

## 9.2 Ansible哪里使用了Jinja2

严格地说，playbook中所有地方都使用了Jinja2，包括几乎所有指令的值、template模板文件、copy模块的content指令的值、lookup的template插件，等等。它们会先经过Jinja2渲染，然后再执行相关任务。

例如，下面的playbook中分别使用了三种Jinja2特殊符号。
```yaml
---
- hosts: localhost
  gather_facts: no
  tasks: 
    - debug: 
        msg: "hello world, {{inventory_hostname}}"
    - debug: 
        msg: "hello world{# comment #}"
    - debug:
        msg: "{% if True %}hello world{% endif %}"
```

> 注：jinja2原生的布尔值应当是小写的true和false，但也支持首字母大写形式的True和False。

执行结果：
```
TASK [debug] ************************
ok: [localhost] => {
    "msg": "hello world, localhost"
}

TASK [debug] ************************
ok: [localhost] => {
    "msg": "hello world"
}

TASK [debug] ************************
ok: [localhost] => {
    "msg": "hello world"
}
```

再比如模板文件a.conf.j2中使用这三种特殊语法：
```jinja2
{# Comment this line #}
variable value: {{inventory_hostname}}
{% if True %}
in if tag code: {{inventory_hostname}}
{% endif %}
```

对应的模板渲染任务：
```yaml
- template:
    src: a.conf.j2
    dest: /tmp/a.conf
```

执行后，将在/tmp/a.conf中生成如下内容：
```
variable value: localhost
in if tag code: localhost
```

有些指令比较特殊，它们已经使用隐式的{% raw %} {{}} {% endraw %}进行了预包围，例如debug模块的var参数、条件判断`when`指令，所以这时就不要手动使用{% raw %} {{}} {% endraw %}再包围指令的值。例如：
```
- debug: 
    var: inventory_hostname
```

但有时候也确实是需要在var或when中的一部分使用{% raw %} {{}} {% endraw %}来包围表示这是一个变量或是一个表达式，而非字符串的。例如：
```
- debug:
    var: hostvars['{{php}}']
  vars:
    - php: 192.168.200.143
```


## 9.3 Jinja2访问元素的两种方式

Jinja2模板引擎允许使用点`.`来访问列表或字典元素，比如`mylist=["a","b","c"]`列表，在Jinja2中既可以使用`mylist[1]`来访问第二个元素，也可以使用`mylist.1`来访问它。

在之前的文章中曾解释过这两种访问方式的区别，这里再重复一遍：  

![](/img/ansible/1625971104444.png)

所以，使用`X.Y`方式时需要小心一些，使用`X["Y"]`更保险。当然，使用哪种方式都无所谓，出错了也知道如何去调整。

## 9.4 Jinja2条件判断

### 9.4.1 if语句块

Jinja2中可以使用if语句进行条件判断。

其语法为：
```jinja2
{% if CONDITION1 %}
  string_or_expression1
{% elif CONDITION2 %}
  string_or_expression2
{% elif CONDITION3 %}
  string_or_expression3
{% else %}
  string_or_expression4
{% endif %}
```

其中elif和else分支都是可省略的。CONDITION部分是条件表达式，关于Jinja2支持的条件表达式，后面会介绍。

例如，模板文件a.txt.j2内容如下：
```jinja2
今天星期几：
{% if whatday == "0" %}
  星期日
{% elif whatday == "1" %}
  星期一
{% elif whatday == "2" %}
  星期二
{% elif whatday == "3" %}
  星期三
{% elif whatday == "4" %}
  星期四
{% elif whatday == "5" %}
  星期五
{% elif whatday == "6" %}
  星期六
{% else %}
  错误数值
{% endif %}
```

上面判断变量whatday的值，然后输出对应的星期几。因为whatday变量的值是字符串，所以让它和字符串形式的数值进行等值比较。当然，也可以使用筛选器将字符串转换为数值后进行数值比较：`whatday|int == 0`。

playbook内容如下：
```
---
- hosts: localhost
  gather_facts: no
  vars_prompt:
    - name: whatday
      default: 0
      prompt: "星期几(0->星期日,1->星期一...):"
      private: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
```

### 9.4.2 行内if表达式

如果if语句的分支比较简单(没有elif逻辑)，那么可以使用行内if表达式。

其语法格式为：
```
string_or_expr1 if CONDITION else string_or_expr2
```

因为行内if是表达式而不是语句块，所以不使用{% raw %} {%%} {% endraw %}符号，而使用{% raw %} {{}} {% endraw %}。

例如：
```yaml
- debug:
    msg: "{{'周末' if whatday|int > 5 else '工作日'}}"
```

## 9.5 for循环

### 9.5.1 for迭代列表

for循环的语法：
```jinja2
{% for i in LIST %}
    string_or_expression
{% endfor %}
```

还支持直接条件判断筛选要参与迭代的元素：
```jinja2
{% for i in LIST if CONDITION %}
    string_or_expression
{% endfor %}
```

此外，Jinja2的for语句还允许使用else分支，如果for所迭代的列表LIST是空列表(或没有元素可迭代)，则会执行else分支。
```jinja2
{% for i in LIST %}
    string_or_expression
{% else %}
    string_or_expression
{% endfor %}
```

例如，在模板文件a.txt.j2中有如下内容：
```jinja2
{% for file in files %}
<{{file}}>
{% else %}
no file in files
{% endfor %}
```

playbook文件内容如下：
```yaml
---
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
      vars: 
        files:
          - /tmp/a1
          - /tmp/a2
          - /tmp/a3
```

执行playbook之后，将生成包含如下内容的/tmp/a.txt文件：
```
</tmp/a1>
</tmp/a2>
</tmp/a3>
```

如果将playbook中的`files`变量设置为空列表，则会执行else分支，所以生成的/tmp/a.txt的内容为：
```
no file in files
```

如果files变量未定义或变量类型不是list，则默认会报错。针对未定义变量，可采用如下策略提供默认空列表：
```jinja2
{% for file in (files|default([])) %}
<{{file}}>
{% else %}
no file in files
{% endfor %}
```

如果不想迭代文件列表中的`/tmp/a3`，则可以加上条件判断：
```jinja2
{% for file in (files|default([])) if file != "/tmp/a3" %}
<{{file}}>
{% else %}
no file in files
{% endfor %}
```

Jinja2的for循环没有提供break和continue的功能，所以只能通过{%raw%}{% for...if...%}{%endraw%}来间接实现类似功能。

### 9.5.2 for迭代字典

默认情况下，Jinja2的for语句只能迭代列表。

如果要迭代字典结构，需要先使用字典的`items()`方法进行转换。如果没有学过python，我下面做个简单解释：

对于下面的字典结构：
```
p: 
  name: junmajinlong
  age: 18
```

如果使用`p.items()`，将计算得到如下结果：
```
[('name', 'junmajinlong'), ('age', 18)]
```

然后for语句中使用两个迭代变量分别保存各列表元素中的子元素即可。下面设置了两个迭代变量key和value：
```
{% for key,value in p.items() %}
```

那么第一轮迭代时，key变量保存的是name字符串，value变量保存的是junmajinlong字符串，那么第二轮迭代时，key变量保存的是age字符串，value变量保存的是18数值。

如果for迭代时不想要key或不想要value，则使用`_`来丢弃对应的值。也可以使用`keys()`方法和`values()`方法分别获取字典的key组成的列表、字典的value组成的列表。例如：
```
{% for key,_ in p.items() %}
{% for _,values in p.items() %}
{% for key in p.keys() %}
{% for value in p.values() %}
```

将上面的解释整理成下面的示例。playbook内容如下：
```yaml
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
      vars:
        p1:
          name: "junmajinlong"
          age: 18
```

模板文件a.txt.j2内容如下：
```jinja2
{% for key,value in p1.items() %}
key: {{key}}, value: {{value}}
{% endfor %}
```

执行结果：
```
key: name, value: junmajinlong
key: age, value: 18
```

### 9.5.3 for的特殊控制变量

在for循环内部，可以使用一些特殊变量，如下：

| Variable           | Description                       |
| :----------------- | :-------------------------------- |
| loop               | 循环本身                           |
| loop.index         | 本轮迭代的索引位，即第几轮迭代(从1开始计数)      |
| loop.index0        | 本轮迭代的索引位，即第几轮迭代(从0开始计数)      |
| loop.revindex      | 本轮迭代的逆向索引位(距离最后一个item的长度，从1开始计数) |
| loop.revindex0     | 本轮迭代的逆向索引位(距离最后一个item的长度，从0开始计数) |
| loop.first         | 如果本轮迭代是第一轮，则该变量值为True                |
| loop.last          | 如果本轮迭代是最后一轮，则该变量值为True              |
| loop.length        | 循环要迭代的轮数，即item的数量                      |
| loop.previtem      | 本轮迭代的前一轮的item值，如果当前是第一轮，则该变量未定义 |
| loop.nextitem      | 本轮迭代的下一轮的item值，如果当前是最后一轮，则该变量未定义 |
| loop.depth         | 在递归循环中，表示递归的深度，从1开始计数 |
| loop.depth0        | 在递归循环中，表示递归的深度，从0开始计数 |
| loop.cycle         | 一个函数，可指定序列作为参数，for每迭代一次便同步迭代序列中的一个元素 |
| loop.changed(*val) | 如果本轮迭代时的val值和前一轮迭代时的val值不同，则返回True |

之前曾介绍过，在Ansible的循环开启`extended`功能之后也能获取一些特殊变量。不难发现，Ansible循环开启`extended`后可获取的变量和此处Jinja2提供的循环变量大多是类似的。所以这里只介绍之前尚未解释过的几个变量。

首先是`loop.cycle()`，它是一个函数，可以传递一个序列(比如列表)作为参数。在for循环迭代时，每迭代一个元素的同时，也会从参数指定的序列中迭代一个元素，如果序列元素迭代完了，则从头开始继续迭代。

例如，playbook内容如下：
```
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
      vars:
        p:
          - aaa
          - bbb
          - ccc
```
模板文件a.txt.j2内容如下：
```jinja2
{% for i in p %}
item: {{i}}
cycle: {{loop.cycle("AAA","BBB")}}
{% endfor %}
```

渲染后得到的/tmp/a.txt文件内容如下：
```
item: aaa
cycle: AAA
item: bbb
cycle: BBB
item: ccc
cycle: AAA
```

然后是`loop.changed(val)`，这也是一个函数。如果相邻的两轮迭代中(即当前一轮和前一轮)，参数val的值没有发生变化，则当前一轮的`loop.changed()`返回False，否则返回True。举个例子很容易理解：

playbook内容如下：
```yaml
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
      vars:
        persons:
          - name: "junmajinlong"
            age: 18
          - name: "junmajinlong"
            age: 22
          - name: "wugui"
            age: 23
```

模板文件a.txt.j2内容如下：
```jinja2
{% for p in persons %}
{% if loop.changed(p.name) %}
index: {{loop.index}}
{% endif %}
{% endfor %}
```

渲染后得到的/tmp/a.txt结果：
```
index: 1
index: 3
```

显然，第二轮迭代时的p.name和前一轮迭代时的p.name值是相同的，所以渲染结果中没有`index: 2`。

## 9.6 Macro

计算机科学当中，Macro(宏)表示的是一段指令的简写，它会在特定的时候替换成其所代表的一大段指令。

如果各位之前不曾知道Macro的概念，我这里用一个不严谨、不属于同一个范畴但最方便大家理解的示例来解释：Shell中的命令别名可以看作是Macro，Shell会在命令开始执行之前(即在Shell解析命令行的阶段)先将别名替换成其原本的值。比如将`ls`替换成`rm -rf`。

Jinja2是支持Macro概念的，宏类似于函数，比如可以接参数，具有代码复用的功能。但其本质和函数是不同的，Macro是替换成原代码片段，函数是直接寻找到地址进行调用。这就不多扯了，好像离题有点远，这可不是编程教程。总的来说，Jinja2的Macro需要像函数一样先定义，在使用的时候直接调用即可，至于底层细节，管它那么多干嘛，又不会涨一毛钱工资。

举一个比较常见的案例，比如某服务的配置文件某指令可以接多个参数值，每个值以空格分隔，每个指令以分号`;`结尾。例如：`log 'host' 'port' 'date';`。如果用模板去动态配置这个指令，可能会使用for循环迭代，但要区分空格分隔符和尾部的分号分隔符。于是，编写如下Macro：
```jinja2
{% macro delimiter(loop) -%}
{{ ' ' if not loop.last else ';' }}
{%- endmacro %}
```

上面表示定义了一个名为delimiter的Macro，它能接一个表示for循环的参数。

上面的Macro定义中还使用了{% raw %} -%} {% endraw %}和{% raw %} {%- {% endraw %}，这是用于处理空白符号的，稍后会解释它的用法，现在各位只需当这个短横线不存在即可。

定义好这个Macro之后，就可以在任意需要的时候去"调用"它。例如：
```jinja2
log {% for item in log_args %}
'{{item}}'{{delimiter(loop)}}
{%- endfor %}

gzip {% for item in gzip_args %}
'{{item}}'{{delimiter(loop)}}
{%- endfor %}
```

提供一个playbook，内容如下：
```yaml
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: a.txt.j2
        dest: /tmp/a.txt
      vars:
        log_args: 
          - host
          - port
          - date
        gzip_args: ['css','js','html']
```

渲染出来的结果如下：
```
log 'host' 'port' 'date';
gzip 'css' 'js' 'html';
```

Macro的参数还可以指定默认值，"调用"Macro并传递参数时，还可以用key=value的方式传递。例如：
```jinja2
{# 定义Macro时，指定默认值 #}
{% macro delimiter(loop,sep=" ",deli=";") -%}
{{ sep if not loop.last else deli }}
{%- endmacro %}

{# "调用"Macro时，使用key=value传递参数值 #}
log {% for item in log_args %}
'{{item}}'{{delimiter(loop,sep=",")}}
{%- endfor %}

gzip {% for item in gzip_args %}
'{{item}}'{{delimiter(loop,deli="")}}
{%- endfor %}
```

渲染得到的结果：
```
log 'host','port','date';
gzip 'css' 'js' 'html'
```

关于Macro，还有些内容可以继续深入(一些变量和call调用的方式)，但应该很少很少用到，所以我这就不再展开了，如果大家有意愿，可以去官方手册学习或网上搜索相关资料，有编程基础的人应该很容易理解，没有编程基础的，就别凑这个热闹了。

## 9.7 block

有些服务程序的配置文件可以使用include指令来包含额外的配置文件，这样可以按不同功能来分类管理配置文件中的配置项。在解析配置文件的时候，会将include指令所指定的文件内容加载并合并到主配置文件中。

Jinja2的block功能有点类似于include指令的功能，block的用法是这样的：先在一个类似于主配置文件的文件中定义block，称为base block或父block，然后在其它文件中继承base block，称为子block。在模板解析的时候，会将子block中的内容填充或覆盖到父block中。

例如，在base.conf.j2文件中定义如下内容：
```jinja2
server {
  listen       80;
  server_name  www.abc.com;

{% block root_page %}
location / {
  root   /usr/share/nginx/html;
  index  index.html index.htm;
}
{% endblock root_page %}

  error_page   500 502 503 504  /50x.html;
{% block err_50x %}{% endblock err_50x %}
{% block php_pages %}{% endblock php_pages %}

}
```

这其实是一个Nginx的虚拟主机配置模板文件。在其中定义了三个block：  
- (1).名为root_page的block，其内部有内容，这个内容是默认内容  
- (2).名为err_50x的block，没有内容  
- (3).名为php_pages的block，没有内容  

如果定义了同名子block，则会使用子block来覆盖父block，如果没有定义同名子block，则会采用默认内容。

下面专门用于定义子block内容的child.conf.j2文件，内容如下：
```jinja2
{% extends 'base.conf.j2' %}

{% block err_50x %}
location = /50x.html {
  root   /usr/share/nginx/html;
}
{% endblock err_50x %}

  {% block php_pages %}
  location ~ \.php$ {
    fastcgi_pass   "192.168.200.43:9000";
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME /usr/share/www/php$fastcgi_script_name;
    include        fastcgi_params;
  }
  {% endblock php_pages %}
```

子block文件中第一行需要使用jinja2的`extends`标签来指定父block文件。这个子block文件中，没有定义名为`root_page`的block，所以会使用父block文件中同名block的默认内容，`err_50x`和`php_pages`则直接覆盖父block。

在template模块渲染文件时，需要指定子block作为其源文件。例如：
```yaml
- hosts: localhost
  gather_facts: no
  tasks: 
    - template:
        src: child.conf.j2
        dest: /tmp/nginx.conf
```

渲染得到的结果:
```
server {
  listen       80;
  server_name  www.abc.com;

location / {
  root   /usr/share/nginx/html;
  index  index.html index.htm;
}

  error_page   500 502 503 504  /50x.html;
location = /50x.html {
  root   /usr/share/nginx/html;
}
  location ~ \.php$ {
    fastcgi_pass   "192.168.200.43:9000";
    fastcgi_index  index.php;
    fastcgi_param  SCRIPT_FILENAME /usr/share/www/php$fastcgi_script_name;
    include        fastcgi_params;
  }
  
}
```

jinja2的block是很出色的一个功能，但在Ansible中应该不太可能用到(或机会极少)，所以多的就不介绍了，有兴趣的可自行找资料了解。

## 9.8 变量赋值和作用域

在Jinja2中也能变量赋值，语法为：
```
{% set var = value %}
{% set var1,var2 = value1,value2 %}
```

例如：
```
{% set mystr = "hello world" %}
{% set mylist1,mylist2 = [1,2,3],[4,5,6] %}
```

如果是在if、for等语句块外进行的变量赋值，则可以在if、for等语句块内引用。例如：
```
{% set mylist = [1,2,3] %}
{% for item in mylist %}
  {{item}}
{% endfor %}
```

但是除if语句块外，其它类型的语句块都有自己的作用域。比如for语句块内部赋值的变量在for退出后就消失。

例如：
```
{% set mylist = [1,2,3] %}
{% set mystr = "hello" %}
{% for item in mylist %}
{{item}}
{%set mystr="world"%}
{% endfor %}
{{mystr}}
```

最后一行渲染的结果是`hello`而不是`world`。

### 9.8.1 如何跨作用域

那如何在for循环内做一个自增操作呢？这应该也是非常常见的需求。但只能说Jinja2里这不方便，只能退而求其次找其它方式，这里我提供两种：
```jinja2
{# 使用loop.index，它本身就是自增的 #}
{% set mylist = [1,2,3] %}
{% for item in mylist %}
name{{loop.index}}
{% endfor %}

{# 使用Jinja2 2.10版的namespace，它可以让变量跨作用域 #}
{% set num = namespace(value=3) %}
{% set mylist = [1,2,3] %}
{% for item in mylist %}
name{{num.value}}
{% set num.value = num.value + 2 %}
{% endfor %}
```

使用上面第二种方案时要注意Jinja2的版本号，Ansible所使用的Jinja2很可能是低于2.10版本的。

## 9.9 Jinja2的空白处理

通常在模板文件中，会将模板代码片段按照编程语言的代码一样进行换行、缩进，但因为它们是嵌套在普通字符串中的，模板引擎并不知道那是一个普通字符串中的空白还是代码格式规范化的空白，而有时候这会带来问题。

比如，模板文件a.txt.j2文件中的内容如下：
```jinja2
line start
line left {% if true %}
  <line1>
{% endif %} line right
line end
```

这个模板文件中的代码部分看上去非常规范，有换行有缩进。一般来说，这段模板文件想要渲染得到的文本内容应该是：
```
line start
line left
<line1>
line right
line end
```

或者是：
```
line start
line left <line1> line right
line end
```

但实际渲染得到的结果：
```
line start
line left   <line1>
 line right
line end
```

渲染的结果中格式很不规范，主要原因就是Jinja2语句块前后以及语句块自身的换行符处理、空白符号处理导致的问题。

Jinja2提供了两个配置项：`lstrip_blocks`和`trim_blocks`，它们的意义分别是：  
- (1).`lstrip_blocks`：设置为true时，会将Jinja2语句块前面的本行前缀空白符号移除  
- (2).`trim_blocks`：设置为true时，Jinja2语句块后的换行符会被移除掉  

对于Ansible的template模块，`lstrip_blocks`默认设置为False，`trim_blocks`默认设置为true。也就是说，默认情况下，template模块会将语句块后面的换行符移除掉，但是会保留语句块前的本行前缀空白符号。

例如，对于下面这段模板片段：
```
line start
    {% if true %}
  <line1>
{% endif %}
line end
```

{% raw %}`{% if`{% endraw %}前的4个空格会保留，{% raw %}`true %}`{% endraw %}后的换行符会被移除，于是{% raw %}'  line1'{% endraw %}(注意前面两个空格)渲染的时候会移到第二行去。再看{% raw %}`{% endif %}`{% endraw %}，{% raw %}`{%`{% endraw %}前面的空白符号会保留，{% raw %}`%}`{% endraw %}后面的换行符会被移除，所以`line end`在渲染时会移动到第三行。第二行和第三行的换行符是由{% raw %}'line1'{% endraw %}这行提供的。

所以结果是：
```
line start
      <line1>
line end
```

一般来说，将`lstrip_blocks`和`trim_blocks`都设置为true，比较符合大多数情况下的空白处理需求。例如：
```yaml
- template:
    src: a.txt.j2
    dest: /tmp/a.txt
    lstrip_blocks: true
    trim_blocks: true
```

渲染得到的结果：
```
line start
  <line1>
line end
```

更符合一般需求的模板格式是，**Jinja2指令部分(比如if、endif、for、endfor等)不要使用任何缩进格式，非Jinja2指令部分按需缩进**。
```
line start
{% if true %}
  <line1>
{% endif %}
line end
```

除了`lstrip_blocks`以及`trim_blocks`可以控制空白外，还可以使用{% raw %}`{%- xxx`{% endraw %}可以移除本语句块前的所有空白(包括换行符)，使用{% raw %}`-%}`{% endraw %}可以移除本语句块后的所有空白(包括换行符)。

注意，`xxx_blocks`这两个配置项和带`-`符号的效果是不同的，总结下：  
- (1).lstrip_blocks只移除语句块前紧连着的且是本行前缀的空白符  
- (2).{% raw %}{%-{% endraw %}移除语句块前所有空白  
- (3).trip_blocks只移除语句块后紧跟着的换行符  
- (4).{% raw %}-%}{% endraw %}`移除语句块后所有的空白  


例如，下面两个模板片段：
```
line1
line2 {%- if true %}
        line3
        line4
      {%- endif %}
line44
line5
```
在两个`xxx_blocks`设置为true时，渲染得到的结果是：
```
line1
line2line3
        line4line44
line5
```

最后想告诉各位，如果渲染后得到的结果是合理的(比如配置文件语法不报错)，就不要追求精确控制空白符号。比如别为了将多个连续的空格压缩成看上去更显规范的单个空格而想方设法(如果你是强迫症，就要小心咯)。如果你还没遇到过这个问题，那以后也肯定会遇到的，其实只要模板稍微写的复杂一点，就能体会到什么叫做"众口难调"。

## 9.10 真实案例：完全自定义的nginx虚拟主机配置

在生产中，一个开发不太完善的系统可能时不时就要去nginx虚拟主机中添加一个location配置段落，如果有多个nginx节点要配置，无疑这是一件让人悲伤的事情。

值得庆幸，Ansible通过Jinja2模板可以很容易地解决这个看上去复杂的事情。

首先提供相关的变量文件`vhost_vars.yml`，内容如下：
```yaml
servers:
  - server_name: www.abc.com
    listen: 80
    locations:
      - match_method: ""
        uri: "/"
        root: "/usr/share/nginx/html/abc/"
        index: "index.html index.htm"
        gzip_types:
          - css
          - js
          - plain

      - match_method: "="
        uri: "/blogs"
        root: "/usr/share/nginx/html/abc/blogs/"
        index: "index.html index.htm"

      - match_method: "~"
        uri: "\\.php$"
        fastcgi_pass: "127.0.0.1:9000"
        fastcgi_index: "index.php"
        fastcgi_param: "SCRIPT_FILENAME /usr/share/www/php$fastcgi_script_name"
        include: "fastcgi_params"

  - server_name: www.def.com
    listen: 8080
    locations:
      - match_method: ""
        uri: "/"
        root: "/usr/share/nginx/html/def/"
        index: "index.html index.htm"

      - match_method: "~"
        uri: "/imgs/.*\\.(png|jpg|jpeg|gif)$"
        root: "/usr/share/nginx/html/def/imgs"
        index: "index.html index.htm"
```

从上面提供的变量文件来看，应该能看出来它的目的是为了能够自动生成一个或多个server段，而且允许随意增删改每个server段中的location及其它指令。这样一来，编写nginx虚拟主机配置的任务就变成了编写这个变量文件。

需注意，每个location段有两个变量名`match_method`和`uri`，作用是生成nginx location配置项的前一部分，即`location METHOD URI {}`。除这两个变量名外，剩余的变量名都会直接当作nginx配置指令渲染到配置文件中，所以它们都需和nginx指令名相同，比如index变量名渲染后会得到nginx的index指令。

剩下的就是写一个Jinja2模板文件，模板中Jinja2语句块标签部分我没有使用缩进，这样比较容易控制格式。文件内容如下：
```jinja2
{# 负责渲染每个指令 #}
{% macro config(key,value) %}
{% if (value is sequence) and (value is not string) and (value is not mapping) %}
{# 如果指令是列表 #}
{% for item in value -%}
{# 如生成的结果是：gzip_types css js plain; #}
{{ key ~ ' ' ~ item if loop.first else item}}{{' ' if not loop.last else ';'}}
{%- endfor %}
{% else %}
{# 如果指令不是列表 #}
{{key}} {{value}};
{% endif %}
{% endmacro %}

{# 负责渲染location指令 #}
{% macro location(d) %}
location {{d.match_method}} {{d.uri}} {
{% for item in d|dict2items if item.key != "match_method" and item.key != "uri" %}
    {{ config(item.key, item.value) }}
{%- endfor %}
  }
{% endmacro %}

{% for server in servers %}
server {
{% for item in server|dict2items %}
{# 非location指令部分 #}
{% if item.key != "locations" %}
  {{ config(item.key,item.value) }}
{%- else %}
{# 各个location指令部分 #}
{% for l in item.value|default([],true) %}
  {{ location(l) }}
{% endfor %}
{% endif %}
{%- endfor %}
}
{% endfor %}
```

然后使用template模块去渲染即可：
```yaml
- hosts: localhost
  gather_facts: no
  vars_files:
    - vhost_vars.yml
  tasks:
    - template:
        src: "vhost.conf.j2"
        dest: /tmp/vhost.conf
```

渲染得到的结果：
```
server {
  server_name www.abc.com;
  listen 80;
  location  / {
    root /usr/share/nginx/html/abc/;
    index index.html index.htm;
    gzip_types css js plain;  }

  location = /blogs {
    root /usr/share/nginx/html/abc/blogs/;
    index index.html index.htm;
  }

  location ~ \.php$ {
    fastcgi_pass 127.0.0.1:9000;
    fastcgi_index index.php;
    fastcgi_param SCRIPT_FILENAME /usr/share/www/php$fastcgi_script_name;
    include fastcgi_params;
  }

}
server {
  server_name www.def.com;
  listen 8080;
  location  / {
    root /usr/share/nginx/html/def/;
    index index.html index.htm;
  }

  location ~ /imgs/.*\.(png|jpg|jpeg|gif)$ {
    root /usr/share/nginx/html/def/imgs;
    index index.html index.htm;
  }

}
```

## 9.11 基本运算符

1.算术类操作符：
```
+
-
*
/
//
%
**
```

说明几点：  
- (1).`+`操作符也可用于字符串串联、列表相加，例如`"a"+"b"`得到"ab"，`[1,2]+[3,4]`得到`[1,2,3,4]`  
- (2).`/`是浮点数除法，例如3/2得到1.5  
- (3).`//`是截断式整除法，例如20/7得到2  
- (4).`*`也可用于重复字符串，例如`"-" * 10`得到10个连续的短横线  

2.比较类操作符：
```
>
<
>=
<=
==
!=
```

需要说明一点：比较操作不仅仅只能比较数值，也能比较其它对象，比如字符串。

例如`"hey" > "hello"`返回True。

3.逻辑运算符：
```
not
and
or
(expr)
```

4.其它操作符：
```
in：成员测试，测试是否在容器内
is：做is测试，参见后文
|：筛选器，参见后文
~：字符串串联
```

需要说明几点：  

![](/img/ansible/1625971248072.png)

## 9.12 Jinja2内置的is测试

jinja2的is操作符可以做很多测试操作，比如测试是否是数值，是否是字符串等等。下表列出了所有Jinja2内置的测试函数。

|               |        |            |          |             |
| :------------ | :----- | :--------- | :------- | :---------- |
| callable()    | even() | le()       | none()   | string()    |
| defined()     | ge()   | lower()    | number() | undefined() |
| divisibleby() | gt()   | lt()       | odd()    | upper()     |
| eq()          | in()   | mapping()  | sameas() | escaped()   |
| iterable()    | ne()   | sequence() |          |             |

其中callable()、escaped()和sameas()在Ansible中几乎用不上，所以不解释。

除了Jinja2的内置测试函数，Ansible还有自己扩展的is测试函数，在后文我会统一列出来。

这些测试函数有些可带参数、有些不带参数，当不带参数或只带一个参数时，括号可以省略。

下面演示两个测试是否是字符串的示例，让大家知道该如何使用这些测试函数进行测试，之后直接解释各测试函数的意义便可。

示例一：在when条件中测试
```yaml
- debug: 
    msg: "a string"
  when: name is string
  vars:
    name: junmajinlong
```

示例二：在jinja2模板的if中测试
```jinja2
{% if name is not string %}
HELLOWORLD
{% endif %}
```

下面是各测试函数的作用：

`defined()`和`undefined()`  
测试变量var是否已定义  

`number()`、`string()`、`none()`  
测试是否是一个数值、字符串、None  

`lt()`、`le()`、`gt()`、`ge()`、`eq()`、`ne()`  
分别测试是否小于、小于等于、大于、大于等于、等于、不等于  

`lower()`和`upper()`  
测试字符串是否全小写、全大写  

`even()`和`odd()`  
测试value是偶数还是奇数  

`divisibleby(num)`  
测试是否能被num整除，例如`18 is divisibleby 3`  

`in(seq)`
测试是否在seq中。例如`3 is in([1,2,3])`、`"h" is in("hey")`  

`mapping()`  
测试是否是一个字典  

`iterable()`  
测试是否可迭代，在Ansible中一般就是list、dict、字符串，当然，在不同场景下可能还会遇到其它可迭代的结构  

`sequence()`  
测试是否是一个序列结构，在Ansible中一般是list、dict、字符串(注：字符串、dict在python中不是序列，但是在Jinja2测试中，属于序列)

纵观上面的内置测试函数，似乎并没有提供直接测试是否是一个列表类型的功能，但在Ansible中却会经常需要去判断所定义的变量是否是一个列表。所以这是一个常见的需求，可参考如下间接测试是否是列表的代码片段：
```
(VAR is sequence) and (VAR is not string) and (VAR is not mapping)
```

如果大家以后深入到Ansible的开发方面，可以自定义Ansible的模块和插件，那么就可以自己写一个更完善的Filter插件来测试是否是list。下面我简单演示下基本步骤，大家能依葫芦画瓢更好，不理解也没任何关系。

首先创建filter_plugins目录并在其内创建一个py文件，例如collection.py，内容如下：
```python
def islist(collection):
  '''
  test data type is a list or not
  '''
  return isinstance(collection, list)

class FilterModule(object):
  '''
  custom jinja2 filter for test list type
  '''
  def filters(self):
    return {
      'islist': islist
    }
```

然后在playbook中便可使用islist()这个筛选器来判断是否是列表类型。例如：
```
- debug: 
    msg: "a list"
  when: p | islist
  vars:
    p:
      - p1
      - p2
```

## 9.13 Ansible扩展的测试函数

模板引擎是多功能的，可以用在很多方面，所以Jinja2自身置的大多数功能都是通用功能。使用Jinja2的工具可能会对Jinja2进行功能扩展，比如Flask扩展了一些功能，Ansible也对Jinja2扩展了一些功能。

Ansible扩展的测试函数官方手册：<https://docs.ansible.com/ansible/latest/user_guide/playbooks_tests.html>。

### 9.13.1 测试字符串

Ansible提供了三个正则测试函数：  
- match()  
- search()  
- regex()  

它们都返回布尔值，匹配成功时返回true。

其中，match()要求从字符串的首字符开始匹配成功。

例如：
```
"hello123world" is match("\d+")    -> False
"hello123world" is match(".*\d+")  -> True
"hello123world" is search("\d+")   -> True
"hello123world" is regex("\d+")    -> True
```

### 9.13.2 版本号大小比较

Ansible作为配置服务、程序的配置管理工具，经常需要比较版本号的大小是否符合要求。Ansible提供了一个`version`测试函数可以用来测试版本号是否大于、小于、等于、不等于给定的版本号。

语法：
```
version('VERSION',CMP)
```

其中CMP可以是如下几种：
```
<, lt, <=, le, >, gt, >=, ge, ==, =, eq, !=, <>, ne
```

例如：
```
{{ ansible_facts["distribution_version"] is version("7.5","<=") }}
```

判断操作系统版本号是否小于等于7.5。

### 9.13.3 子集、父集测试

- `A is subset(B)`测试A是否是B的子集  
- `A is superset(B)`测试A是否是B的父集  

例如：
```
- debug:
    msg: '{{[1,2,3] is subset([1,2,3,4])}}'
```

### 9.13.4 成员测试

Jinja2自己有一个`in`操作符可以做成员测试，Ansible另外还实现了一个contains测试函数，主要目的是为了结合select、reject、selectattr和rejectattr筛选器。

官方给了一个示例：
```
vars:
  lacp_groups:
    - master: lacp0
      network: 10.65.100.0/24
      gateway: 10.65.100.1
      dns4:
        - 10.65.100.10
        - 10.65.100.11
      interfaces:
        - em1
        - em2

    - master: lacp1
      network: 10.65.120.0/24
      gateway: 10.65.120.1
      dns4:
        - 10.65.100.10
        - 10.65.100.11
      interfaces:
          - em3
          - em4

tasks:
  - debug:
      msg: "{{ (lacp_groups|selectattr('interfaces', 'contains', 'em1')|first).master }}"
```

此外，Ansible还实现了`all`和`any`测试函数，`all()`测试表示当序列中所有元素都返回true时，all()返回true，`any()`测试表示当序列中只要有元素返回true，any()就返回true。

仍然是官方给的示例：
```
  mylist:
      - 1
      - "{{ 3 == 3 }}"
      - True
  myotherlist:
      - False
      - True
tasks:
  - debug:
      msg: "all are true!"
    when: mylist is all

  - debug:
      msg: "at least one is true"
    when: myotherlist is any
```

### 9.13.5 测试文件

Ansible提供了测试文件的相关函数：  
- is exists：是否存在  
- is directory：是否是目录  
- is file：是否是普通文件  
- is link：是否是软链接  
- is abs：是否是绝对路径  
- is same_file(F)：是否和F是硬链接关系  
- is mount：是否是挂载点  

```yaml
- debug:
    msg: "path is a directory"
  when: mypath is directory

# 如果mypath是绝对路径，即is测试返回true，
# 则筛选器返回absolute，否则返回relative
- debug:
    msg: "path is {{ (mypath is abs)|ternary('absolute','relative')}}"

- debug:
    msg: "path is the same file as path2"
  when: mypath is same_file(path2)

- debug:
    msg: "path is a mount"
  when: mypath is mount
```

### 9.13.6 测试任务的执行状态

每个任务的执行结果都有4种状态：成功、失败、changed、跳过。

Ansible提供了相关的测试函数：  
- succeeded、success  
- failed、failure  
- changed、change  
- skipped、skip 

```yaml
- shell: /usr/bin/foo
  register: result
  ignore_errors: True

- debug:
    msg: "it failed"
  when: result is failed

- debug:
    msg: "it changed"
  when: result is changed

- debug:
    msg: "it succeeded in Ansible >= 2.1"
  when: result is succeeded

- debug:
    msg: "it succeeded"
  when: result is success

- debug:
    msg: "it was skipped"
  when: result is skipped
```

## 9.14 Jinja2内置Filter

通常，模板语言都会带有筛选器，JinJa2也不例外，每个筛选器函数都是一个功能，作用就类似于函数，而且它也可以接参数。

Jinja2的筛选器使用方式非常简单，直接使用一根竖线`|`，在模板解析时，Jinja2会将竖线左边的返回值或计算结果当作隐式参数传递给竖线右边的筛选器函数。另外，筛选器是一个表达式，所以写在{% raw %} {{}} {% endraw %}内部。

例如，Jinja2有一个内置lower()筛选器函数，可以将字符串全部转化成小写字母。
```yaml
- debug:
    msg: "{{'HELLO WORLD'|lower()}}"
```

如果筛选器函数没有给定参数，则括号可以省略，例如`"HELLO"|lower`。

有些筛选器函数需要给定参数，例如replace()筛选器，可以将字符串中的一部分替换掉。例如，将字符串中的"no"替换成"yes"。
```yaml
{% if result %}
{{result|replace('no', 'yes')}}
{%endif%}
```

JinJa2内置了50多个筛选器函数，Ansible自身也扩展了一些方便的筛选器函数，所以数量非常多。如下：

|                  |               |              |              |             |
| :--------------- | :------------ | :----------- | :----------- | :---------- |
| abs()            | float()       | lower()      | round()      | tojson()    |
| attr()           | forceescape() | map()        | safe()       | trim()      |
| batch()          | format()      | max()        | select()     | truncate()  |
| capitalize()     | groupby()     | min()        | selectattr() | unique()    |
| center()         | indent()      | pprint()     | slice()      | upper()     |
| default()        | int()         | random()     | sort()       | urlencode() |
| dictsort()       | join()        | reject()     | string()     | urlize()    |
| escape()         | last()        | rejectattr() | striptags()  | wordcount() |
| filesizeformat() | length()      | replace()    | sum()        | wordwrap()  |
| first()          | list()        | reverse()    | title()      | xmlattr()   |

我会将它们中绝大多数的含义列举出来(剩下一部分是我觉得在Ansible中用不上的，比如escape()转义为HTML安全字符串)，各位没必要全都测试一遍，但是速看一遍并大概了解它们的含义和作用是有必要的。

(1).`float(default=0.0)`  
将数值形式的字符串转换为浮点数。如果无法转换，则返回默认值0.0。可使用default参数自定义转换失败时的默认值。

例如`"abcd"|float`、`""|float`都转换为0.0，`""|float('NaN')`返回的是字符串NaN，表示非数值含义。

(2).`int(default=0,base=10)`  
将数值形式的字符串直接截断为整数。如果无法转换，则返回默认值0。可使用default参数自定义转换失败时的默认值。

此外，还可以指定进制参数base，比如base=2表示将传递过来的参数当作二进制进行解析，然后转换为10进制数值。

例如`'3.55'|int`结果为3，`'0b100'|int(base=2)`结果为4。

(3).`abs()`  
计算绝对值。

注意，只能计算数值，如果传递的是字符串，可使用筛选器int()或float()先转换成数值。例如`'-3.14'|float|abs`。

(4).`round(precision=0,method='common')`  
对数值进行四舍五入。第一个参数指定四舍五入的精度，第二个参数指定四舍五入的方式，有三种方式可选择：  
- (1).ceil：只入不舍  
- (2).floor：只舍不入  
- (3).common：小于五的舍，大于等于5的入  

注意：  
- (1).只能计算数值，如果传递的是字符串，可使用筛选器int()或float()先转换成数值  
- (2).计算的是整数，则返回值是整数，计算的是浮点数，则返回值是浮点数  

例如`42.55|round`的结果为43.0，`45|round`的结果是45，`42.55|round(1,'floor')`的结果是42.5。

(5).`random()`  
返回一个随机整数。竖线左边的值X决定了随机数的范围为`[0,X)`。例如`5|random`生成的随机数可能是0、1、2、3、4。

(6).`list()`  
转换为列表。如果要转换的目标是字符串，则返回的列表是字符串中的每个字符。

例如`range(1,4)|list`的结果是`[1,2,3]`，`"hey"|list`的结果是`["h","e","y"]`。

(7).`string()`  
转换为字符串。

例如`"{{333|string+'aa'}}"`结果为"333aa"。

(8).`tojson()`  
转换为json格式。

(9).`lower()`、`upper()`、`title()`、`capitalize()`  
lower()将大写字母转换为小写。upper()将小写字母转换为大写。title()将每个首字母转为大写。capitalize()将第一个单词首字母转为大写。

(10).`min()`、`max()`  
从序列中取最小、最大值。

例如`["a","abddd","cba"]|max`得到cba。

(11).`sum(start=0)`  
计算序列中各元素的算术和。可指定start参数作为算术和的起点。

例如`[1,2,3]|sum`得到6，`[1,2,3]|sum(start=3)`得到9。

(12).`trim()`  
移除字符串前缀和后缀空白。

例如`"  abcd    "|trim ~ "DEF"`得到"abcdDEF"。

(13)`truncate()`  
截断字符串为指定长度。主要用于web编程，Ansible用不到。

(14).`replace(old,new,count=None)`  
将字符串中的old替换成new，count参数指定替换多少次，默认替换所有匹配成功的。

例如`"you see see you"|replace("see","look")`得到`you look look you`，而`replace("see","look",1)`则得到`you look see you`。

(15).`first()`、`last()`  
返回序列中的第一个、最后一个元素。

例如`"hello world" | last`返回字母d，`[2,3,4]|last`返回数值4。

(16).`map(attribute='xxx')`  
如果一个列表中包含了多个dict，map可根据指定的属性名(即dict的key)，从列表中各dict内筛选出该属性值部分。

例如，对于如下变量：
```
p:
  - name: "junma"
    age: 23
  - name: woniu
    age: 22
    weight: 45
  - name: tuner
    age: 25
    weight: 50
```

`p|map(attribute="name")|list`将得到`["junma","woniu","tuner"]`。

(17).`select()`、`reject()`  
从序列中选中、排除满足条件的项。

例如：
```
{{ numbers|select("odd") }}         ->选出奇数
{{ numbers|select("even") }}        ->选出偶数
{{ numbers|select("lt", 42) }}      ->选出小于42的数
{{ strings|select("eq", "mystr") }} ->选出"mystr"元素
{{ numbers|select("divisibleby", 3) }} ->选出被3整除的数
```

其中测试参数可以指定为支持的测试函数，在前文已经介绍过。

(18).`selectattr()`、`rejectattr()`  
根据对象属性筛选、排除序列中的多个元素。这个有时候很好用。

比如：
```
p:
  - name: "junma"
    age: 23
  - name: woniu
    age: 22
    weight: 45
  - name: tuner
    age: 25
    weight: 50
```

筛选所有age大于22岁的对象：
```
p|selectattr('age','gt',22)|list
```
得到的结果：
```
[
  {"age": 23,"name": "junma"},
  {"age": 25,"name": "tuner","weight": 50}
]
```

(19).`batch(count,fill_withs=None)`  
将序列中每count个元素打包成一个列表。最后一个列表可能元素个数不够，默认不填充，如果要填充，则指定`fill_with`参数。

例如`[1,2,3,4,5]|batch(2)|list`得到`[[1,2],[3,4],[5]]`，`[1,2,3,4,5]|batch(2,'x')|list`得到`[[1,2],[3,4],[5,'x']]`。

(20).`default('default_value',bool=False)`或`d()`  
如果竖线左边的变量未定义，则返回default()指定的默认值。默认只对未定义变量其作用，如果想让default()也能对布尔类型的数据生效，需将第二个参数设置为true。

`d()`是`default()`的简写方式。

例如`myvar|d('undefined')`在myvar不存在时返回undefined字符串，`""|d("empty")`中因为是空字符串而不是未定义变量，所以仍然返回空字符串，`""|d("empty",true)`则返回empty字符串。

(21).`unique()`  
对序列中进行去重操作。

例如`[1,2,3,3,1,2]|unique`得到结果`[1,2,3]`。

(22).`join(d="")`  
将序列中的元素使用d参数指定的符号串联成字符串，默认连接符为空字符串。

例如`[1,2,3]|join("-")`得到`1-2-3`，`[1,2,3]|join`得到123。

(23).`length()`和`count()`  
返回序列中元素的数量或字符串的字符个数。length和count是别名等价的关系。

(24).`wordcount`  
计算字符串中的单词个数。

(25).`reverse()`  
颠倒序列元素。

例如`"hello"|reverse`得到`olleh`，`[1,2,3]|reverse|list`得到`[3,2,1]`。

(26).`filesizeformat(binary=False)`  
将数值大小转换为kB、MB、GB单位。默认按照1000为单位进行换算，如果指定binary参数为True，则按1024进行换算。

(27).`slice(N, fill_with=None)`  
将序列均分为N个列表，可指定`fill_with`参数来填充均分时不足的列表。

例如：
```
[1,2,3,4,5]|slice(3)    |list -> [[1,2],[3,4],[5]]
[1,2,3,4]  |slice(3,"x")|list -> [[1,2],[3,"x"],[4,"x"]]
```

(28).`groupby(attribute)`  
根据指定的属性对dict进行分组。看下面的例子：

```yaml
- debug:
    msg: '{{person|groupby("name")}}'
  vars:
    person:
      - name: "junma"
        age: 23
      - name: "junma"
        age: 33
      - name: woniu
        age: 22
        weight: 45
      - name: tuner
        age: 25
        weight: 50
```

得到的结果：
```
[
  ["junma", [{"age": 23,"name": "junma"},{"age": 33,"name": "junma"}]],
  ["tuner", [{"age": 25,"name": "tuner","weight": 50}]],
  ["woniu", [{"age": 22,"name": "woniu","weight": 45}]]
]
```

(29).`indent(width=4)`  
对字符串进行缩进格式化。默认缩进宽度为4。

(30).`format()`  
类似于`printf`的方式格式化字符串。对于熟悉Python的人来说，使用`%`或`str.format()`格式化字符串要更方便些。

下面是等价的：
```
{{ "%s, %s!"|format("hello", "world") }}
{{ "%s, %s!" % ("hello", "world") }}
{{ "{}, {}!".format("hello", "world") }}
```

(31).`center(witdth=80)`  
将字符串扩充到指定宽度并放在中间位置，默认80个字符。

例如`"hello"|center(10)|replace(" ","-")`得到`"--hello---"`。

(32).`sort(reverse=False, case_sensitive=False, attribute=None)`  
对序列中的元素进行排序，attribute指定按照什么进行排序。

例如`["acd","a","ca"]|sort`得到`["a","acd","ca"]`。

再例如，对于下面的变量：
```
person:
  - name: "junma"
    age: 23
  - name: woniu
    age: 22
    weight: 45
  - name: tuner
    age: 25
    weight: 50
```

可以使用`person|sort(attribute="age")`对3个元素按照age进行升序排序。

(33).`dictsort(case_sensitive=False, by='key', reverse=False)`  
对字典进行排序。默认按照key进行排序，可以指定为`value`按照value进行排序。

例如：
```
person:
  p2:
    name: "junma"
    age: 23
  p1:
    name: woniu
    age: 22
    weight: 45
  p3:
    name: tuner
    age: 25
    weight: 50
```

可以使用`person|dictsort`按照key(即p1、p2、p3)进行排序，结果是先p1，再p2，最后p3。

## 9.15 Ansible扩展的Filter

Ansible扩展了非常多的Filter，非常非常多，本来我只想介绍一部分。但是想到有些人不愿看英文，我还是将它们全都写出来，各位权当看中文手册好了。实际上它们也都非常容易，绝大多数筛选器用法几乎都不用动脑，一看便懂。

### 9.15.1 类型转换类筛选器

例如：
```
{{"123"|int}}
{{"123"|float}}
{{123|string}}
{{range(1,6)|list}}
{{123|bool}}
```

注意，没有dict筛选器转换成字典类型。

### 9.15.2 获取当前时间点

Ansible提供的now()可以获取当前时间点。

例如：
```yaml
- debug:
    msg: '{{now()}}'
```

得到结果：
```
ok: [localhost] => {
  "msg": "2020-01-25 00:27:17.563627"
}
```

可以指定输出的格式化字符串，支持的格式化字符串参考python官方手册：<https://docs.python.org/3/library/datetime.html#strftime-strptime-behavior>。

例如：
```yaml
- debug:
    msg: '{{now().strftime("%Y-%m-%d %H:%M:%S.%f")}}'
```

### 9.15.3 YAML、JSON格式化

Ansible提供了几个和YAML、JSON格式化相关的Filter：
```
to_yaml
to_json
to_nice_yaml
to_nice_json
```

它们都可使用indent参数指定缩进的层次。

`to_yaml`和`to_json`适用于调试，`to_nice_yaml`和`to_nice_json`适用于用户查看。

例如：
```
- debug:
    msg: '{{f1|to_nice_json(indent=2)}}'
  vars:
    f1:
      father: "Bob"
      mother: "Alice"
      Children:
        - Judy
        - Tedy
```

### 9.15.4 参数忽略

Ansible提供了一个特殊变量omit，可以用来忽略模块的参数效果。

官方手册给了一个非常有代表性的示例，如下：
```yaml
- name: touch files with an optional mode
  file:
    dest: "{{ item.path }}"
    state: touch
    mode: "{{ item.mode | default(omit) }}"
  loop:
    - path: /tmp/foo
    - path: /tmp/bar
    - path: /tmp/baz
      mode: "0444"
```

当所迭代的元素中不存在mode项，则使用默认值，默认值设置为特殊变量omit，使得file模块的mode参数被忽略，相当于未书写该参数。只有给定了mode项时，mode参数才生效。

### 9.15.5 列表元素连接

`join`可以将列表各个元素根据指定的连接符连接起来：
```
{{ [1,2,3] | join("-") }}
```

### 9.15.6 列表压平

前面的文章曾介绍过flatten筛选器，它可以将嵌套列表压平。

例如：

```yaml
- debug:
    msg: "{{ [3, [4, 2] ] | flatten }}"
- debug:
    msg: "{{ [3, [4, [2]] ] | flatten(levels=1) }}"
```

### 9.15.7 并集、交集、差集

Ansible提供了集合理论类的求值操作：  
- unique：去重  
- union：并集，即两个集合中所有元素  
- intersect：交集，即两个集合中都存在的元素  
- difference：差集，即返回只在第一个集合中，不在第二个集合中的元素  
- symmetric_difference：对称差集，即返回两个集合中不重复的元素  

```
- name: return [1,2,3]
  debug:
    msg: "{{ [1,2,3,2,1] | unique }}"
- name: return [1,2,3,4]
  debug:
    msg: "{{ [1,2,3] | union([2,3,4]) }}"
- name: return [2,3]
  debug:
    msg: "{{ [1,2,3] | intersect([2,3,4]) }}"
- name: return [1]
  debug:
    msg: "{{ [1,2,3] | difference([2,3,4]) }}"
- name: return [1,4]
  debug:
    msg: "{{ [1,2,3] | symmetric_difference([2,3,4]) }}"
```

### 9.15.8 dict和list转换

- `dict2items()`：将字典转换为列表  
- `items2dict()`：将列表转换为字典  

对于`dict2items`，例如：
```
- debug:
    msg: "{{ p | dict2items }}"
  vars:
    p:
      name: junmajinlong
      age: 28
```

得到：
```
[
  {
    "key": "name",
    "value": "junmajinlong"
  },
  {
    "key": "age",
    "value": 28
  }
]
```

对于`items2dict`，例如：
```
- debug:
    msg: "{{ p | items2dict }}"
  vars:
    p:
      - key: name
        value: junmajinlong
      - key: age
        value: 28
```

得到：
```
{
  "age": 28,
  "name": "junmajinlong"
}
```

默认情况下，`dict2items`和`items2dict`都使用`key`和`value`来转换，但它们都允许使用`key_name`和`value_name`自定义转换的名称。

例如：
```yaml
- debug:
    msg: "{{  files | dict2items(key_name='file', value_name='path') }}"
  vars:
    files:
      users: /etc/passwd
      groups: /etc/group
```
得到：
```
[
  {
    "file": "users",
    "path": "/etc/passwd"
  },
  {
    "file": "groups",
    "path": "/etc/group"
  }
]
```

### 9.15.9 zip和zip_longest

`zip`和`zip_longest`可以将多个列表的元素一一对应并组合起来。它们的区别是，zip以短序列为主，`zip_longest`以最长序列为主，缺失的部分使用null填充。

例如：
```yaml
- name: return [[1,"a"], [2,"b"]]
  debug:
    msg: "{{ [1,2] | zip(['a','b']) | list }}"

- name: return [[1,"a"], [2,"b"]]
  debug:
    msg: "{{ [1,2] | zip(['a','b','c']) | list }}"

- name: return [[1,"a"], [2,"b"], [null, "c"]]
  debug:
    msg: "{{ [1,2] | zip_longest(['a','b','c']) | list }}"

- name: return [[1,"a","aa"], [2,"b","bb"]]
  debug:
    msg: "{{ [1,2] | zip(['a','b'], ['aa','bb']) | list }}"
```

在Python中经常会将zip的运算结果使用dict()构造成字典，Jinja2中也可以：
```
- name: !unsafe 'return {"a": 1, "b": 2}'
  debug:
    msg: "{{ dict(['a','b'] | zip([1,2])) }}"
```

### 9.15.10 子元素subelements

subelements筛选器在前一章节详细解释过，这里不再介绍。

### 9.15.11 random生成随机数

Jinja2自身内置了一个random筛选器，Ansible也有一个random筛选器，比Jinja2内置的定制性要更强一点。

```
"{{ ['a','b','c'] | random }}"
# => 'c'

# 生成[0,60)的随机数
"{{ 60 | random }} * * * * root /script/from/cron"
# => '21 * * * * root /script/from/cron'

# 生成[0,100)的随机数，步进为10
{{ 101 | random(step=10) }}
# => 70

# 生成[1,100)的随机数，步进为10
{{ 101 | random(1, 10) }}
# => 31
{{ 101 | random(start=1, step=10) }}
# => 51

# 指定随机数种子。
# 下面指定为主机名，所以同主机生成的随机数相同，但不同主机的随机数不同
"{{ 60 | random(seed=inventory_hostname) }} * * * * root /script/from/cron"
```

### 9.15.12 shuffle打乱顺序

![](/img/ansible/1625971347318.png)

### 9.15.13 json_query

可查询Json格式的数据，`json_query`在Ansible中非常实用，是必学Filter之一。

Ansible的`json_query`基于jmespath，所以需要先安装jmespath：
```shell
$ pip3 install jmespath
```

jmespath的查询语法相关示例可参见其官方手册：  
- 入门手册： <http://jmespath.org/tutorial.html>  
- 示例：<http://jmespath.org/examples.html>  

下面我列出Ansible中给出的示例。

例如，对于下面的数据结构：
```
{
  "domain_definition": {
    "domain": {
      "cluster": [
        {"name": "cluster1"},
        {"name": "cluster2"}
      ],
      "server": [
        {
          "name": "server11",
          "cluster": "cluster1",
          "port": "8080"
        },
        {
          "name": "server12",
          "cluster": "cluster1",
          "port": "8090"
        },
        {
          "name": "server21",
          "cluster": "cluster2",
          "port": "9080"
        },
        {
          "name": "server22",
          "cluster": "cluster2",
          "port": "9090"
        }
      ],
      "library": [
        {
          "name": "lib1",
          "target": "cluster1"
        },
        {
          "name": "lib2",
          "target": "cluster2"
        }
      ]
    }
  }
}
```

使用
```
{{ domain_definition | json_query('domain.cluster[*].name') }}
```
可以获取到名称cluster1和cluster2。

使用
```
{{ domain_definition | json_query('domain.server[*].name') }}
```
可以获取到server11、server12、server21和server22。

使用
```
- debug:
    var: item
  loop: "{{ domain_definition | json_query(server_name_cluster1_query) }}"
  vars:
    server_name_cluster1_query: "domain.server[?cluster=='cluster1'].port"
```

可以迭代8080和8090两个端口。

注意上面使用了问号`?`，这表示后面的是一个表达式。

使用
```
"{{domain_definition | json_query('domain.server[?cluster==`cluster2`].{name1: name, port1: port}')}}"
```


可得到：
```
[
  {
    "name1": "server21",
    "port1": "9080"
  },
  {
    "name1": "server22",
    "port1": "9090"
  }
]
```

注意上面使用了反引号`` ` ` ``而不是单双引号，因为单双引号都被使用过了，再使用就不方便，可读性也差。

### 9.15.14 ip地址筛选

Ansible提供了非常丰富的功能来完成IP地址的筛选，用法非常多，绝大多数关于IP、网络地址类的计算、查询需求都能解决。

使用它需要先安装python的netaddr包：
```shell
$ pip3 install netaddr
```

完整用法参考官方手册：<https://docs.ansible.com/ansible/latest/user_guide/playbooks_filters_ipaddr.html#playbooks-filters-ipaddr>。

下面是我整理的一部分用法。

检测是否是合理的IP地址：
```
{{ myvar | ipaddr }}
```

检测是否是合理的ipv4地址、ipv6地址：
```
{{ myvar | ipv4 }}
{{ myvar | ipv6 }}
```

从列表中筛选出合理的IP地址：
```
test_list = ['192.24.2.1', 'host.fqdn', '::1', '192.168.32.0/24', 'fe80::100/10', True, '']

# {{ test_list | ipaddr }}
['192.24.2.1', '::1', '192.168.32.0/24', 'fe80::100/10']

# {{ test_list | ipv4 }}
['192.24.2.1', '192.168.32.0/24']

# {{ test_list | ipv6 }}
['::1', 'fe80::100/10']
```

获取IP地址部分：
```
{{ '192.0.2.1/24' | ipaddr('address') }}
{{ ipvar | ipv4('address') }}
{{ ipvar | ipv6('address') }}
```

检测或找出公网IP和私网IP：
```
# {{ test_list | ipaddr('public') }}
['192.24.2.1']

# {{ test_list | ipaddr('private') }}
['192.168.32.0/24', 'fe80::100/10']
```

### 9.15.15 正则表达式筛选器

Ansible提供了几个正则类的Filter，主要有：  
- `regex_search`：普通正则匹配  
- `regex_findall`：全局匹配  
- `regex_replace`：正则替换  

例如：
```
{{ 'foobar' | regex_search('(foo)') }}

# 匹配失败时返回空
{{ 'ansible' | regex_search('(foobar)') }}

# 多行模式、忽略大小写的匹配
{{ 'foo\nBAR' | regex_search("^bar", multiline=True, ignorecase=True) }}

# 全局匹配
# 每次匹配到的内容将存放在一个列表中
{{ 'DNS servers 8.8.8.8 and 8.8.4.4' | regex_findall('\\b(?:[0-9]{1,3}\\.){3}[0-9]{1,3}\\b') }}

# 正则替换
# "ansible" 替换为 "able"
{{ 'ansible' | regex_replace('^a.*i(.*)$', 'a\\1') }}

# "foobar" 替换为 "bar"
{{ 'foobar' | regex_replace('^f.*o(.*)$', '\\1') }}

# 使用命名捕获，"localhost:80" 替换为 "localhost, 80"
{{ 'localhost:80' | regex_replace('^(?P<host>.+):(?P<port>\\d+)$', '\\g<host>, \\g<port>') }}

# "localhost:80" 替换为 "localhost"
{{ 'localhost:80' | regex_replace(':80') }}
```

### 9.15.16 URL处理筛选器

`urlsplit`筛选器可以从一个URL中提取fragment、hostname、netloc、password、path、port、query、scheme、以及username。如果不传递任何参数，则直接返回一个包含了所有字段的字典。

```
{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('hostname') }}
# => 'www.acme.com'

{{ "http://user:password@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('netloc') }}
# => 'user:passwd@www.acme.com:9000'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('username') }}
# => 'user'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('password') }}
# => 'passwd'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('path') }}
# => '/dir/index.html'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('port') }}
# => '9000'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('scheme') }}
# => 'http'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('query') }}
# => 'query=term'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit('fragment') }}
# => 'fragment'

{{ "http://user:passwd@www.acme.com:9000/dir/index.html?query=term#fragment" | urlsplit }}
# =>
#   {
#       "fragment": "fragment",
#       "hostname": "www.acme.com",
#       "netloc": "user:password@www.acme.com:9000",
#       "password": "passwd",
#       "path": "/dir/index.html",
#       "port": 9000,
#       "query": "query=term",
#       "scheme": "http",
#       "username": "user"
#   }
```

### 9.15.17 编写注释的筛选器

在模板渲染中，有可能需要在目标文件中生成一些注释信息。Ansible提供了`comment`筛选器来完成该任务。

```
{{ "Plain style (default)" | comment }}
```

会得到：
```
#
# Plain style (default)
#
```

可以自定义注释语法：
```
{{ "My Special Case" | comment(decoration="! ") }}
```
得到：
```
!
! My Special Case
!
```

### 9.15.18 extract提取元素

extract结合map使用时，可以根据索引(列表索引或字典索引)从列表或字典中提取对应元素的值。

```
{{ [0,2] | map('extract', ['x','y','z']) | list }}
{{ ['x','y'] | map('extract', {'x': 42, 'y': 31}) | list }}
```
得到：
```
['x', 'z']
[42, 31]
```

### 9.15.19 dict合并

combine筛选器可以将多个dict中同名key进行合并(以覆盖的方式合并)。

```
{{ {'a':1, 'b':2} | combine({'b':3}) }}
```
得到：
```
{'a':1, 'b':3}
```
使用`recursive=True`参数，可以递归到嵌套dict中进行覆盖合并：
```
{{ {'a':{'foo':1, 'bar':2}, 'b':2} | combine({'a':{'bar':3, 'baz':4}}, recursive=True) }}
```
将得到：
```
{'a':{'foo':1, 'bar':3, 'baz':4}, 'b':2}
```
可以合并多个dict，如下：
```
{{ a | combine(b, c, d) }}
```
d中同名key会覆盖c，c会覆盖b，b会覆盖a。

### 9.15.20 hash值计算

计算字符串的sha1：
```
{{ 'test1' | hash('sha1') }}
```

计算字符串的md5：
```
{{ 'test1' | hash('md5') }}
```

计算字符串的checksum，默认即`hash('sha1')`值：
```
{{ 'test2' | checksum }}
```

计算sha512密码(随机salt):

```
{{ 'passwd' | password_hash('sha512') }}
```

计算sha256密码(指定salt):
```
{{ passwd' | password_hash('sha256', 'mysalt') }}
```

同一主机生成的密码相同：
```
{{ 'passwd' | password_hash('sha512', 65534 | random(seed=inventory_hostname) | string) }}
```

根据字符串生成UUID值：
```
{{ hostname | to_uuid }}
```

### 9.15.21 base64编解码筛选器

```
{{ encoded_str | b64decode }}
{{ decoded_str | b64encode }}
```

例如，Ansible有一个`slurp`模块，它的作用类似于fetch模块，它可以从目标节点中读取指定文件的内容，然后以base64方式编码返回，所以要获取其原始内容，需要base64解码。

例如：
```yaml
- slurp:
    src: "/var/run/sshd.pid"
  register: sshd_pid
- debug: 
    msg: "base64_pid: {{sshd_pid.content}}"
- debug: 
    msg: "sshd_pid: {{sshd_pid.content|b64decode}}"
```

结果：
```
TASK [slurp] ***************
ok: [localhost]

TASK [debug] ******************
ok: [localhost] => {
    "msg": "base64_pid: MTE4OAo="
}

TASK [debug] *****************
ok: [localhost] => {
    "msg": "base64_pid: 1188\n"
}
```

### 9.15.22 文件名处理

- `basename`：获取字符串中的文件名部分  
- `dirname`：获取字符串中目录名部分  
- `expanduser`：扩展家目录，即将`~`替换为家目录  
- `realpath`：获取软链接的原始路径  
- `splitext`：扩展名分离  

对于splitext筛选器，例如：
```
{{"nginx.conf"|splitext}}
#=> ("nginx",".conf")

{{'/etc/my.cnf'|splitext}}
#=> ("/etc/my",".cnf")
```

### 9.15.23 日期时间类处理

相对来说，Ansible中处理日期时间的机会是比较少的。但是Ansible也提供了比较方便的处理日期时间的方式。

使用`strftime`将当前时间或给定时间(只能以epoch数值指定)按照给定的格式输出：
```
# 将当前时间点以year-month-day hour:min:sec格式输出
{{ '%Y-%m-%d %H:%M:%S' | strftime }}

# 将指定的时间按照指定格式输出
{{ '%Y-%m-%d' | strftime(0) }}          # => 1970-01-01
{{ '%Y-%m-%d' | strftime(1441357287) }} # => 2015-09-04
```

使用`to_datetime`可以将日期时间字符串转换为Python日期时间对象，既然得到了对象，就可以进行时间比较、时间运算等操作。
```
# 计算时间差(单位秒)
# 默认解析的日期时间字符串格式为%Y-%m-%d %H:%M:%S，但可以自定义格式
{{ (("2016-08-14 20:00:12" | to_datetime) - ("2015-12-25" | to_datetime("%Y-%m-%d"))).total_seconds() }}
#=>20203212.0

# 计算相差多少天。只考虑天数，不考虑时分秒等
{{ (("2016-08-14 20:00:12" | to_datetime) - ("2015-12-25" | to_datetime('%Y-%m-%d'))).days  }}
```

### 9.15.24 human\_to\_bytes和human\_readable

`human_readable`将数值转换为人类可读的数据量大小单位：
```yaml
{{1|human_readable}}
#=> 1.00 Bytes
{{1|human_readable(isbits=True)}} 
#=>1.00 bits
{{10240|human_readable}}
#=>10.00 KB
{{102400000|human_readable}}
#=>97.66 MB
{{102400000|human_readable(unit="G")}}
#=>0.10 GB
{{102400000|human_readable(isbits=True, unit="G")}}
#=>0.10 Gb
```

`human_to_bytes`将人类可读的单位转换为数值：
```
{{'0'|human_to_bytes}}        #= 0
{{'0.1'|human_to_bytes}}      #= 0
{{'0.9'|human_to_bytes}}      #= 1
{{'1'|human_to_bytes}}        #= 1
{{'10.00 KB'|human_to_bytes}} #= 10240
{{   '11 MB'|human_to_bytes}} #= 11534336
{{  '1.1 GB'|human_to_bytes}} #= 1181116006
{{'10.00 Kb'|human_to_bytes(isbits=True)}} #= 10240
```

### 9.15.25 其它筛选器

`quote`为字符串加引号，比如方便编写shell模块的命令：
```yaml
- shell: echo {{ string_value | quote }}
```

`ternary`根据true、false来决定返回哪个值：
```yaml
# gender为male时，返回Mr，否则返回Ms
{{ (gender == "male") | ternary('Mr','Ms') }}

# 根据true、false、null来决定
{{ enabled | ternary('no shutdown', 'shutdown', omit) }}
```

`product`生成笛卡尔积：
```
{{['foo', 'bar'] | product(['com'])|list}}
#=>[["foo","com"], ["bar","com"]]
```

## 9.16 Python对象自身的方法

在前面解释过，当使用`x.y`的方式访问y的时候，会先寻找x的y属性或y方法，找不到才开始找Jinja2中定义的属性。

所以，在Ansible中有些时候是可以直接使用Python对象自身方法的，比如字符串对象可以使用`endswith`判断字符串是否以某字符串结尾。

这也为Ansible提供了非常有用的功能。但是有些人可能没学过Python，所以也不知道有哪些方法可用，也不理解有些代码是什么作用。这一点我也没有办法帮助各位，但大家也不用太过在意，几个方法而已，细节罢了。事实上也就字符串对象的方法比较多。

### 9.16.1 Python字符串对象方法

在Jinja2中，经常会使用到字符串。如何使用字符串对象的方法？

例如，Python字符串对象有一个upper方法，可以将字符串改变为大写字母，直接使用`"abc".upper()`，注意不要省略小括号，这一点和Jinja2和Shell函数都是不一样的。

例如：
```yaml
- debug:
    msg: '{{ "abc".upper() }}'
- debug:
    msg: "{{ 'foo bar baz'.upper().split() }}"
```

得到：
```
TASK [debug] ****************
ok: [localhost] => {
    "msg": "ABC"
}

TASK [debug] *****************
    "msg": ["FOO", "BAR", "BAZ"]
}
```

下面是字符串对象的各种方法，我简单说明了它们的功能，关于它们的用法和示例可参见我的博客：<https://www.junmajinlong.com/python/string_methods/>。

```
lower：将字符串转换为小写字母
upper：将字符串转换为大写字母
title：将字符串所有单词首字母大写，其它字母小写
capitalize：将字符串首单词首字母大写，其它字母小写
swapcase：将字符串中所有字母大小写互换
isalpha：判断字符串中所有字符是否是字母
isdecimal：判断字符串中所有字符是否是数字
isdigit：判断字符串中所有字符是否是数字
isnumeric：判断字符串中所有字符是否是数字
isalnum：判断字符串中所有字符是否是字母或数字
islower：判断字符串中所有字符是否是小写字母
isupper：判断字符串中所有字符是否是大写字母
istitle：判断字符串中是否所有单词首字符大写，其它小写
isspace：判断字符串中所有字符是否是空白字符(空格、制表符、换行符等)
isprintable：判断字符串中所有字符是否是可打印字符(如制表符、换行符不是可打印字符)
isidentifier：判断字符串中是否符合标识符定义规则(即只包含字母、数字或下划线，且字母或下划线开头)
center：在左右两边填充字符串到指定长度，字符串居中
ljust：在右边填充字符串到指定长度
rjust：在左边填充字符串到指定长度
zfill：使用0填充在字符串左边
count：计算字符串中某字符或某子串出现的次数
endswith：字符串是否以某字符或某子串结尾
startswith：字符串是否以某字符或某子串开头
find：从左向右检查字符串中是否包含某子串，搜索不到时返回-1
rfind：从右向左检查字符串中是否包含某子串，搜索不到时返回-1
index：功能类似于find，搜索不到时报错
rindex：功能类似于rfind，搜索不到时报错
replace：替换字符串
expandtabs：将字符串中的制表符\t替换成空格，默认替换为8空格
split：将字符串分割得到列表
rsplit：从右向左分割字符串得到列表
splitlines：按行分割字符串，每行作为列表的一个元素
join：将列表各元素使用指定符号连接起来，例如`"-".[1,2,3]`得到`1-2-3`
strip：移除字符串前缀和后缀指定字符，如果没有指定移除的字符，则默认移除空白
lstrip：移除字符串指定的前缀字符，如果没有指定移除的字符，则默认移除空白
rstrip：移除字符串指定的后缀字符，如果没有指定移除的字符，则默认移除空白
format：格式化字符串，此外还可使用Python的`%`格式化方式，如{{ '%.2f' % 1.2345 }}
```

### 9.16.2 list对象方法

虽然Python中list对象有很多操作方式，但应用到Ansible中，大概也就两个个方法值得了解：  
- `count()`：(无参数)计算列表中元素的个数或给定参数在列表中出现的次数  
- `index()`：检索给定参数在列表中的位置，如果元素不存在，则报错  

例如：
```yaml
- debug:
    msg: '{{ [1,2,1,1,3,4].count() }}'
- debug:
    msg: '{{ [1,2,1,1,3,4].count(1) }}'
- debug:
    msg: '{{ [1,2,1,1,3,4].index(2) }}'
```
