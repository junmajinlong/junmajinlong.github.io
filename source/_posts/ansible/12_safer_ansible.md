---
title: 12.让Ansible更安全：使用Vault进行加密
p: ansible/12_safer_ansible.md
date: 2021-07-11 10:37:41
tags: Ansible
categories: Ansible
---

--------

**回到：[Ansible系列文章](/ansible/index)**  

--------

> 各位读者，请您：由于Ansible使用Jinja2模板，它的模板语法{% raw %} {{}} {% endraw %}和{% raw %} {%%} {% endraw %}和我博客系统hexo的模板使用的符号一样，在渲染时会产生冲突，尽管我尽我努力地花了大量时间做了调整，但无法保证已经全部都调整。因此，如果各位阅读时发现一些明显的诡异的错误(比如像这样的空的` `行内代码)，请一定要回复我修正这些渲染错误。

# 12.让Ansible更安全：使用Vault进行加密

管理目标节点时，有些操作需要使用密码才允许访问，但Ansible是一个自动化配置管理工具，在自动化操作的阶段中要求交互式输入密码的行为应该是一件让人败兴的事。

通常，实现非交互式的方案有：

- (1).将敏感数据写入文件(比如写入变量文件)，然后读取，这种方案不安全；  
- (2).定义敏感数据对应的环境变量，缺点是有些客户端工具不一定支持这种方式，且这种方案不够方便；  
- (3).使用命令行选项，但缺点是不安全，且Ansible不支持这种功能；  
- (4).expect类工具，缺点是不方便。  

Ansible自身提供了更方便更安全的方案：Vault加密，它使用AES256加密算法为Ansible保驾护航。

在Ansible 2.4版本以前，其Vault加密功能并不友好，功能限制较大，比如所有任务涉及到的加密必须共享使用同一个密码，比如更早的版本不能加密部分字符串。现在的Ansible版本的Vault加密功能则比较友好，本文会详细介绍友好版本的Vault加密功能。

## 12.1 一个入门示例：创建一个加密文件

Ansible使用ansible-valut命令完成Vault加解密相关操作，它有很多子命令，比如创建加密文件的create子命令、查看加密文件的view子命令，等等，这些子命令的选项大多类似。

```shell
$ ansible-vault --help

positional arguments:
  {create,decrypt,edit,view,encrypt,encrypt_string,rekey}
    create              创建新的文件并加密
    decrypt             解密已加密的文件
    edit                编辑已加密文件的内容
    view                查看已加密文件的内容
    encrypt             加密已存在的未加密文件
    encrypt_string      加密一段字符串
    rekey               修改已加密文件的Vault ID和凭据密码
```

例如，使用create子命令创建一个加密文件`passwd_prompt.yml`，并在此文件中写入一个密码变量。

```shell
$ ansible-vault create --vault-id @prompt passwd_prompt.yml

New vault password (default):             # 提示用户输入的
Confirm new vault password (default):     # 提示用户输入的

---                      # 自动打开编辑器，比如打开vim
mypasswd: 123456         # 输入一个变量
                         # 保存并退出
```

上面使用了`--vault-id @prompt`作为ansible-vault的参数，当执行`ansible-vault create`时，这会以交互式的方式提示用户输入一个密码，这个密码是加密这个加密文件`passwd_prompt.yml`所需的密码。简单地说，这个文件里保存了加密的密码数据，但要访问这个文件，也需要密码。为了区分，**本文之后将称该密码为"凭据密码"**。

在上面的示例中，我设置了一个变量`mypasswd`，且其值为`123456`。当退出编辑器后，会自动加密该文件。例如：

```shell
# 为排版需求，我截断了一部分字符
$ cat passwd_prompt.yml
$ANSIBLE_VAULT;1.1;AES256
316336393534333831363334363633653232303268
373962616633356439393830353430373230613239
323530393466363035633164306133663030303839
6137616561353337660a3430313931376266643034
373234623839383838386537643635666232303563
```

如果想要查看该密文，可使用ansible-vault的view子命令，同样需要提供凭据密码。

```shell
$ ansible-vault view --vault-id @prompt passwd_prompt.yml
Vault password (default):      # 输入凭据密码后将展示如下内容
---
mypasswd: 123456
```

另外，请暂时记住加密文件中的第一行`$ANSIBLE_VAULT;1.1;AES256`，稍后会解释该行各字段的含义。

## 12.2 Vault ID和凭据密码提供方式

在上面的示例中，使用了选项`--vault-id @prompt`，它的行为是以交互式的方式提示用户输入凭据密码。

实际上，`--valut-id`选项的完整用法是这样的：

```
--vault-id label@prompt
--vault-id label@path_to_normal_txt_file
--vault-id label@path_to_script_file
```

在解释这些用法之前，介绍一点Ansible Vault的历史：在以前的Ansible版本中，每次执行ansbile或ansible-playbook命令时都只能提供一个凭据密码，这使得本次执行的所有任务中所涉及到的所有加密都只能共同使用这一个凭据密码。

![](/img/ansible/1625971774190.png)

```
--vault-id dev_mysql@xxx
--vault-id test_mysql@xxx
--vault-id prod_mysql@xxx
```

也可以省略Vault ID，这时将统一使用默认的Vault ID。

`--vault-id label@xxx`中的xxx是什么呢？它是凭据密码的来源。凭据密码的来源有三种：  

- (1).prompt：prompt是固定的值，表示以交互式的方式提示用户提供凭据密码  
- (2).普通文件的文件名：比如"a.txt"，表示从该文件中读取凭据密码  
- (3).脚本文件名：表示从该脚本执行结果中获取凭据密码  

例如有普通文件a.txt：

```shell
echo '123456' >a.txt
```

有脚本文件a.sh内容如下(py脚本、perl脚本甚至是程序等都可以，只要密码会输出到标准输出并且有可执行权限即可)：

```shell
#!/bin/bash
echo 123456
```

那么下面用三种不同的凭据密码源来创建三个加密文件：

```shell
$ ansible-vault create --vault-id id1@prompt first_encrypted.yml
$ ansible-vault create --vault-id id2@a.txt  second_encrypted.yml
$ ansible-vault create --vault-id id3@a.sh   third_encrypted.yml
```

如果省略Vault ID，则：

```shell
$ ansible-vault create --vault-id @prompt first_encrypted1.yml

$ ansible-vault create --vault-id a.txt  second_encrypted1.yml
$ ansible-vault create --vault-id @a.txt  second_encrypted1.yml

$ ansible-vault create --vault-id a.sh   third_encrypted1.yml
$ ansible-vault create --vault-id @a.sh   third_encrypted1.yml
```

当需要访问加密数据时，比如ansible命令、ansible-playbook命令、ansible-vault命令等，需指定与加密时相同的Vault ID，且可以指定多个。

例如：

```shell
# 以文件的方式获取凭据密码
ansible-vault view --vault-id id2@a.txt  second_encrypted.yml

# 以交互式方式提供凭据密码
ansible-vault view --vault-id id2@prompt  second_encrypted.yml

# 指定多个凭据密码，通常用在ansible-playbook命令中，
# 比如变量文件中多个敏感数据使用了不同的vault id加密
ansible-playbook --vault-id id1@a.txt --vault-id id2@a.sh main.yml
```

## 12.3 加密已存在的文件

如果想要加密一个已经存在的文件，使用`ansible-vault encrypt`，用法和create子命令几乎一样。

例如，plain.yml文件目前是明文的变量文件：

```shell
$ cat plain.yml
---
plain_passwd: 123456
port: 2312
```

加密之：

```shell
$ ansible-vault encrypt --vault-id id1@a.txt plain.yml
Encryption successful

$ cat plain.yml
$ANSIBLE_VAULT;1.2;AES256;id1
336130623037643739633938313061336431
363838333864616665623034336439316234
643337663235366365316332643063653863
3534303533633334340a6438326433616236
383566346533653135313633633139613133
353935653734363833616464326662336132
```

可以一次性加载多份数据，但是它们使用相同的Vault ID和密码。

```shell
$ ansible-vault encrypt --vault-id idx@a.txt foo.yml bar.yml baz.yml
```

## 12.4 加密协议头的含义

在每个已加密的文件中，都包含了一行头部数据。可使用cat命令查看，例如：

```shell
$ cat plain.yml
$ANSIBLE_VAULT;1.2;AES256;id1        #<==加密协议头
336130623037643739633938313061336431
363838333864616665623034336439316234
643337663235366365316332643063653863
3534303533633334340a6438326433616236
383566346533653135313633633139613133
353935653734363833616464326662336132
```

有两种协议头：

```
$ANSIBLE_VAULT;1.1;AES256
$ANSIBLE_VAULT;1.2;AES256;id1
```

其中第一个字段目前只能是`$ANSIBLE_VAULT`。

第二个字段1.1、1.2表示的是Vault格式的版本号，如果给定了Vault ID，则版本号为1.2，如果使用默认的Vault ID，则版本号为1.1。老版本还有1.0版本的，但现在不会设置成该版本号，1.0版本号的加密数据只能被读。

第三个字段目前只支持AES256加密算法。

第四个字段，如果加密时指定了Vault ID，则第四个字段保存的是Vault ID，否则不设置第四个字段。

当忘记了某个加密文件对应的Vault ID后，可以直接cat命令查看该加密文件来获取ID，也可以直接提取出来：

```shell
$ awk -F';' 'NR==1{print $4}' encrypted.yml
id2
```

如果Ansible需要访问多个Vault加密的文件，将会自动根据Vault ID去寻找用户提供的凭据密码。

例如，对于如下命令来说，main.yml中引用了两个加密的变量文件，那么自然需要两个Vault ID和两个密码。当任务执行过程中需要解密第一个文件时，Ansible将自动获取其Vault ID并从命令行中寻找对应的Vault ID及其凭据密码来源。

```shell
$ ansible-playbook --vault-id id1@a.txt --vault-id id2@b.txt main.yml 
```

## 12.5 解密已加密的文件

对于一个已Vault加密的文件，可使用`ansible-vault decrypt`解密文件。

例如，解密上面示例中已加密的plain.yml：

```shell
$ ansible-vault decrypt --vault-id id1@a.sh plain.yml
Decryption successful

$ cat plain.yml
---
plain_passwd: 123456
port: 2312
```

解密多个文件：

```shell
$ ansible-vault decrypt --vault-id idx@a.txt foo.yml bar.yml baz.yml
```

## 12.6 修改Vault ID和凭据密码

对于一个已Vault加密的文件，可使用`ansible-vault rekey`修改其Vault ID或其凭据密码。

例如，`first_passwd.yml`是一个已加密的文件：

```shell
$ cat first_encrypted.yml
$ANSIBLE_VAULT;1.2;AES256;id1
6361623434363834363065643838346361383830
3038366239353535663330653463376666346636
6439663030643839623965613563343632343234
3739313638363262390a62666466343864626466
3235633862306366383161663765663962306132
```

修改它的Vault ID和凭据密码：

```shell
$ ansible-vault rekey --vault-id id1@a.txt \
                      --new-vault-id id2@b.txt \
                      first_encrypted.yml
Rekey successful
```

再看加密后的内容，无论是Vault ID还是密文都已经改变了：

```shell
$ cat first_encrypted.yml
$ANSIBLE_VAULT;1.2;AES256;id2
366666343166313633396135346464653537373936
343936653437616235316630333538343139336361
656330646535656335376662623062343863306561
3033383533306266620a3866396139663235303931
313230666235303833633330653863323738623339
```

当然，也可以只修改Vault ID或只修改凭据密码，只需保持`label@xxx`中的其中一项不变即可。例如：

```shell
# Vault ID不变，只修改凭据密码
$ ansible-vault rekey --vault-id id1@a.txt \
                      --new-vault-id id1@b.txt \
                      first_encrypted.yml
                      
# 凭据密码，只修改Vault ID
$ ansible-vault rekey --vault-id id1@a.txt \
                      --new-vault-id id2@a.txt \
                      first_encrypted.yml
```

## 12.7 编辑已加密文件的内容

如果想要编辑一个已Vault加密的文件内容，使用`ansible-vault edit`命令。

```shell
$ ansible-vault edit --vault-id id2@a.txt first_encrypted.yml
```

它将自动打开默认的编辑器(如vim)让用户进行编辑。内部过程是先创建一个带有原始明文数据的临时文件(在打开vim时，vim底部状态栏看到临时文件名)，用户修改的操作都在该临时文件中进行，当保存时，将自动加密并覆盖原文件。

也可以采用先解密、再修改、再重新加密的方式，而且这种方式可以以非交互式的方式进行。

```shell
ansible-vault decrypt xxx
sed -i 'xx' xxx
ansible-vault encrypt xxx
```

## 12.8 ansible-playbook任务中使用Vault加密文件

对数据进行加密的原始目标是在playbook文件、task文件或变量文件中隐藏敏感数据。那么使用加密的文件呢？

例如，名为test.yml的playbook文件内容如下：

```yaml
---
- hosts: localhost
  gather_facts: no
  vars_files: 
    - first_passwd.yml
    - second_passwd.yml
  tasks: 
    - name: debug var in first_passwd
      debug: 
        var: passwd1
    - name: debug var in second_passwd
      debug: 
        var: passwd2
```

此playbook中引入了两个变量文件`first_passwd.yml`和`second_passwd.yml`，然后分别用两个任务去访问这两个变量文件中定义的两个变量passwd1和passwd2。

这两个变量文件的内容如下：

```yml
# first_passwd.yml内容：
---
passwd1: "passwd in first_passwd"

# second_passwd.yml内容：
---
passwd2: "passwd in second_passwd"
```

使用不同的Vault ID和凭据密码加密这两个文件：

```shell
$ echo 'abcdef' >a.txt
$ echo 'ABCDEF' >b.txt

$ ansible-vault encrypt --vault-id passwd1@a.txt first_passwd.yml
$ ansible-vault encrypt --vault-id passwd2@b.txt second_passwd.yml
```

然后执行该playbook，因为涉及到了多个加密文件的访问，且Vault ID不同，凭据密码也不同，所以需要指定多个`--vault-id`选项。如下：

```shell
$ ansible-playbook --vault-id passwd1@a.txt --vault-id passwd2@b.txt test.yml
......
TASK [debug var in first_passwd] *********************
ok: [localhost] => {
    "passwd1": "passwd in first_passwd"
}

TASK [debug var in second_passwd] ********************
ok: [localhost] => {
    "passwd2": "passwd in second_passwd"
}
......
```

## 12.9 加密字符串并嵌入YAML文件

使用`ansible-vault encrypt_string`可以加密一段字符串。

例如：

```shell
$ ansible-vault encrypt_string --vault-id id1@a.txt 'hello' --name 'mysql_pass' 
mysql_pass: !vault |
          $ANSIBLE_VAULT;1.2;AES256;id1
          39623437656130313338613033383464376437
          66303938343635623430353664303334623138
          33353435366262653437663839316261623664
          6362633233653237360a343439376437633733
          6262
Encryption successful
```

其中`hello`是需要被加密的明文数据，`--name mysql_pass`是yaml中的key，比如可以作为变量的名称。也可以省略`--name`选项。

```shell
$ ansible-vault encrypt_string --vault-id id1@a.txt 'hello'
!vault |
          $ANSIBLE_VAULT;1.2;AES256;id1
          61336334663365656361316532356133623
          62343863366463396338333933343635373
          39656461643962643831653561313266366
          6432316334653339340a336565333361346
          3635
Encryption successful
```

加密一段字符串之后，就可以将这段加密后的字符串原样拷贝到yaml文件中。比如下面是MySQL Role的一个变量文件：

```yaml
---
mysql_port: 3306
mysql_user: root
mysql_pass: !vault |
          $ANSIBLE_VAULT;1.2;AES256;id1
          39623437656130313338613033383464376437
          66303938343635623430353664303334623138
          33353435366262653437663839316261623664
          6362633233653237360a343439376437633733
          6262
mysql_host: 192.168.200.27
```

此外，`ansible-vault encrypt_string`也可以从标准输入中获取要加密的明文数据：

```shell
$ ansible-vault encrypt_string --vault-id id1@a.txt --stdin-name 'mysql_pass' <<<"hello"
mysql_pass: !vault |
          $ANSIBLE_VAULT;1.2;AES256;id1
          35386538393939633862626361323735306
          63383966356265333336303231313735376
          38346566373262663866363631373539313
          6531643131306537350a363232656661303
          6333
Encryption successful

# 或者
$ echo -n 'hello' | ansible-vault encrypt_string --vault-id id1@a.txt --stdin-name 'mysql_pass'
```

注意从标准输入获取待加密数据时，使用的选项是`--stdin-name`，而不是`--stdin --name`。

## 12.10 加速加解密的过程

如果要加解密的文件较多，按照官方文档所说，可安装cryptography包来解决。

```
pip install cryptography
```

## 12.11 Vault加密最佳实践

Ansible Vault可以加密所有的yaml、json格式的文件，但是不建议直接加密一个包含很多数据的文件，因为加密后不方便查看和修改任务文件，建议加密只包含敏感数据的文件。

比如，下面这个变量文件：

```yaml
---
mysql_port: 3306
mysql_user: root
mysql_pass: xxxxxxxx
mysql_host: 192.168.200.27
```

其实该文件只有`mysql_pass`是想要隐藏的敏感数据，建议的做法是在此文件中使用jinja2再引用一次以`vault_`开头的变量，以`vault_`开头是为了一眼就能看到这是Vault加密的变量。例如：

```yaml
---
mysql_port: 3306
mysql_user: root
mysql_pass: "{{vault_mysql_pass}}"
mysql_host: 192.168.200.27
```

然后单独在一个文件中定义被引用的变量`vault_mysql_pass`并加密该变量文件。例如，在`mysql_pass.yml`文件里定义`vault_mysql_pass`变量：

```yaml
---
vault_mysql_pass: "abcdef"
```

加密之：

```shell
$ ansible-vault encrypt --vault-id mysql@a.txt mysql_pass.yml
```
