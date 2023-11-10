---
title: 手动生成/etc/shadow文件中的密码
p: linux/make_shadow_passwd.md
date: 2020-08-07 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 手动生成/etc/shadow文件中的密码

shadow文件的格式就不说了。就说说它的第二列——密码列。

通常，passwd直接为用户指定密码就ok了。但在某些情况下，要为待创建的用户事先指定密码，还要求是加密后的密码，例如kickstart文件中的rootpw指令，ansible创建用户时提前指定密码等，这时候不得不手动生成合理的密码。

先说说shadow文件中第二列的格式，它是加密后的密码，它有些玄机，不同的特殊字符表示特殊的意义：

```
1.该列留空，即"::"，表示该用户没有密码。
2.该列为"!"，即":!:"，表示该用户被锁，被锁将无法登陆，但是可能其他的登录方式是不受限制的，如ssh公钥认证的方式，su的方式。
3.该列为"*"，即":*:"，也表示该用户被锁，和"!"效果是一样的。
4.该列以"!"或"!!"开头，则也表示该用户被锁。
5.该列为"!!"，即":!!:"，表示该用户从来没设置过密码。
6.如果格式为"$id$salt$hashed"，则表示该用户密码正常。其中$id$的id表示密码的加密算法，$1$表示使用MD5算法，$2a$表示使用Blowfish算法，"$2y$"是另一算法长度的Blowfish,"$5$"表示SHA-256算法，而"$6$"表示SHA-512算法，
```

目前基本上都使用sha-512算法的，但无论是md5还是sha-256都仍然支持。`$salt$`是加密时使用的salt，hashed才是真正的密码部分。

下文都以生成明文"123456"对应的加密密码为例。

要生成md5/sha256/sha512算法的密码，使用openssl即可。

```
openssl passwd -1 '123456'
openssl passwd -1 -salt 'abcdefg' '123456'
```

生成密码后，直接将其拷贝或替换到shadow文件的第二列即可。例如：替换root用户的密码

```
shell> field=$(awk -F ':' '/^root/{print $2}' /etc/shadow)
shell> password=$(openssl passwd -1 123456)
shell> sed -i '/^root/s%'$field'%'$password'%' /etc/shadow
```

但老版本的openssl passwd不支持生成sha-256和sha-512算法的密码。在CentOS 6上，可以借助grub提供的密码生成工具grub-crypt生成。

```
[root@server1 ~]# grub-crypt -h
Usage: grub-crypt [OPTION]...
Encrypt a password.

  -h, --help              Print this message and exit
  -v, --version           Print the version information and exit
  --md5                   Use MD5 to encrypt the password
  --sha-256               Use SHA-256 to encrypt the password
  --sha-512               Use SHA-512 to encrypt the password (default)

Report bugs to <bug-grub@gnu.org>.
EOF
[root@server1 ~]# grub-crypt --sha-512
Password: 
Retype password: 
$6$nt4hMDAYqYjudvfo$AKIZ3Z0o6/6HV6GKXqq21VEmh.ADFAZUQw2mvbIlplKx7gu9MQiEWjdmHnF2YPnYzgce1cP/bzDguVnUkMg/N.
```

grub-crypt其实是一个python脚本，交互式生成密码。以下是grub-crypt文件的内容。

```python
[root@server1 ~]# cat /sbin/grub-crypt 
#! /usr/bin/python

'''Generate encrypted passwords for GRUB.'''

import crypt
import getopt
import getpass
import sys

def usage():
    '''Output usage message to stderr and exit.'''
    print >> sys.stderr, 'Usage: grub-crypt [OPTION]...'
    print >> sys.stderr, 'Try `$progname --help\' for more information.'
    sys.exit(1)

def gen_salt():                      # 生成随机的salt
    '''Generate a random salt.'''
    ret = ''
    with open('/dev/urandom', 'rb') as urandom:
        while True:
            byte = urandom.read(1)
            if byte in ('ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz'
                        './0123456789'):
                ret += byte
                if len(ret) == 16:
                    break
    return ret

def main():
    '''Top level.'''
    crypt_type = '$6$' # SHA-256
    try:
        opts, args = getopt.getopt(sys.argv[1:], 'hv',
                                   ('help', 'version', 'md5', 'sha-256',
                                    'sha-512'))
    except getopt.GetoptError, err:
        print >> sys.stderr, str(err)
        usage()
    if args:
        print >> sys.stderr, 'Unexpected argument `%s\'' % (args[0],)
        usage()
    for (opt, _) in opts:
        if opt in ('-h', '--help'):
            print (
'''Usage: grub-crypt [OPTION]...
Encrypt a password.

  -h, --help              Print this message and exit
  -v, --version           Print the version information and exit
  --md5                   Use MD5 to encrypt the password
  --sha-256               Use SHA-256 to encrypt the password
  --sha-512               Use SHA-512 to encrypt the password (default)

Report bugs to <bug-grub@gnu.org>.
EOF''')
            sys.exit(0)
        elif opt in ('-v', '--version'):
            print 'grub-crypt (GNU GRUB 0.97)'
            sys.exit(0)
        elif opt == '--md5':
            crypt_type = '$1$'
        elif opt == '--sha-256':
            crypt_type = '$5$'
        elif opt == '--sha-512':
            crypt_type = '$6$'
        else:
            assert False, 'Unhandled option'
    password = getpass.getpass('Password: ')
    password2 = getpass.getpass('Retype password: ')
    if not password:
        print >> sys.stderr, 'Empty password is not permitted.'
        sys.exit(1)
    if password != password2:
        print >> sys.stderr, 'Sorry, passwords do not match.'
        sys.exit(1)
    salt = crypt_type + gen_salt()
    print crypt.crypt(password, salt)      # 生成最终的加密密码

if __name__ == '__main__':
    main()
```

很不幸，CentOS 7上默认安装的是grub2，它不提供grub-crypt。因此参照grub-crypt内容，使用下面的python语句简单代替grub-crypt，这同样也是交互式的。

```
python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
```

如果不想交互式，再改成如下形式：

```
python -c 'import crypt,getpass;pw="123456";print(crypt.crypt(pw))'
```

现在就方便多了，直接将结果赋值给变量即可。

```
[~]# a=$(python -c 'import crypt,getpass;pw="123456";print(crypt.crypt(pw))')
[~]# echo $a
$6$uKhnBg5A4/jC8KaU$scXof3ZwtYWl/6ckD4GFOpsQa8eDu6RDbHdlFcRLd/2cDv5xYe8hzw5ekYCV5L2gLBBSfZ.Uc166nz6TLchlp.
```

例如，ansible创建用户并指定密码：

```
a=$(python -c 'import crypt,getpass;pw="123456";print(crypt.crypt(pw))')
ansible  192.168.100.55 -m user -a 'name=longshuai5 password="$a" update_password=always'
```