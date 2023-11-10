---
title: openssl命令用法详解
p: linux/openssl_subcmds.md
date: 2021-03-15 15:17:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# openssl命令用法详解

openssl命令的格式是"openssl cmd cmd-optns args"，cmd是openssl的子命令，官方手册上也称为伪命令(pseudo-command)。openssl有很多种子命令，每个子命令实现各自的功能，大部分cmd都可以直接man command查看命令的用法和功能。

本文并没有介绍所有子命令的用法，其中关于tls/ssl证书相关的子命令(比如req、ca等)和相关内容，参考[openssl申请、颁发证书和自建CA](/linux/openssl_ca)。

## openssl有哪些子命令

以下是openssl支持的子命令，常用命令或可能用的上的命令加粗加红显示了，这些命令的用户在后面的文章中会一一介绍。

```
[root@xuexi ~]# openssl --help
openssl:Error: '--help' is an invalid command.
 
# 支持的标准命令
Standard commands
asn1parse   ca        ciphers    cms       crl
crl2pkcs7  dgst       dh         dhparam
dsa        dsaparam   ec         ecparam
enc        engine     errstr     gendh
gendsa     genpkey    genrsa     nseq
ocsp       passwd     pkcs12     pkcs7
pkcs8      pkeyparam  pkey       pkeyutl
prime      rand       req        rsa
rsautl     s_client   s_server   s_time
sess_id    smime      speed      spkac
ts         verify     version    x509
 
# 指定"dgst"命令时即单向加密支持的算法，实际上支持更多的算法，具体见dgst命令
Message Digest commands (see the `dgst' command for more details)
md2   md4  md5  rmd160  sha  sha1             

# 指定对称加密"enc"时支持的对称加密算法
Cipher commands (see the `enc' command for more details)
camellia-128-cbc  camellia-128-ecb  camellia-192-cbc  camellia-192-ecb 
camellia-256-cbc  camellia-256-ecb  cast   cast-cbc
aes-128-cbc  aes-128-ecb   aes-192-cbc   aes-192-ecb      
aes-256-cbc  aes-256-ecb   base64        bf               
bf-cbc       bf-cfb        bf-ecb        bf-ofb           
cast5-cbc    cast5-cfb     cast5-ecb     cast5-ofb        
des          des-cbc       des-cfb       des-ecb          
des-ede      des-ede-cbc   des-ede-cfb   des-ede-ofb      
des-ede3     des-ede3-cbc  des-ede3-cfb  des-ede3-ofb     
des-ofb      des3          desx          idea             
idea-cbc     idea-cfb      idea-ecb      idea-ofb         
rc2          rc2-40-cbc    rc2-64-cbc    rc2-cbc          
rc2-cfb      rc2-ecb       rc2-ofb       rc4              
rc4-40       seed          seed-cbc      seed-cfb         
seed-ecb     seed-ofb      zlib             
```

看上去非常复杂？其实不复杂，只是子命令多点而已，而且很多子命令经常用到的选项也就1到两个。

<a name="blogpasswd"></a>

以下是各子命令的选项`-passin`和`-passout`可能使用到的密码传递格式，`-passin`指的是传递解密时的密码，`-passout`指的是传递加密输出文件时的密码。如果不给定密码格式，将提示从终端输入。这一点在后面的文章中将不再细述。

```
格式一：pass:password   ：password表示传递的明文密码
格式二：env:var         ：从环境变量var获取密码值
格式三：file:filename   ：filename文件中的第一行为要传递的密码。若filename同时传递给"-passin"和"-passout"选项，则filename的第一行为"-passin"的值，第二行为"-passout"的值
格式四：stdin           ：从标准输入中获取要传递的密码
```
例如，要加密某个密钥文件，使得每次使用该密钥文件都需要输入密码，则使用`-passout`指定加密密码，当使用被加密的密钥文件时需要解密，使用`-passin`传递解密密码。

## openssl genrsa

genrsa用于生成RSA私钥，**不会生成公钥，因为公钥提取自私钥**，如果需要查看公钥或生成公钥，可以使用[openssl rsa](#openssl_rsa_pkey)命令。

使用man genrsa查询其用法。
```
openssl genrsa [-out filename] [-passout arg] [-des] [-des3] [-idea] [numbits]

选项说明：
-out filename ：将生成的私钥保存至filename文件，若未指定输出文件，则为标准输出。
-numbits      ：指定要生成的私钥的长度，默认为1024。该项必须为命令行的最后一项参数。
-passout args ：加密私钥文件时，传递密码的格式，如果要加密私钥文件时单未指定该项，则提示输入密码。
-des|-des3|-idea：指定加密私钥文件用的算法，这样每次使用私钥文件都将输入密码，太麻烦所以很少使用。
```

其中，`-passout args`传递密码的args的格式见[openssl密码格式](#blogpasswd)。

例如：

**(1).生成512位的rsa私钥，输出到屏幕。**

```
[root@xuexi tmp]# openssl genrsa 512
Generating RSA private key, 512 bit long modulus
..................++++++++++++
.........++++++++++++
e is 65537 (0x10001)
-----BEGIN RSA PRIVATE KEY-----
MIIBOwIBAAJBAMDIpW+o0E/XLUHB9XYgNcbLuiAA/wToy0v3GEIdtycWiK1ikXfo
XYf2PCd5ynFtNl7D8jXekr2wgnySNostXSkCAwEAAQJAdMmYnzovaA681fdAUl1U
9qd4i+bOlxTIA68fPP5vc/d+Kqk+fQlHsTmXo0ZpvdLig1v+5EsjoOlbNqkyjTVC
YQIhAPXuK5rPHlYIcmcSWrJNmmACSUz8NZb56cS/4Uq0f289AiEAyK1laGNVGIsg
ROLJqgTRP7vHHhXcQj3RhX9G/3oJBF0CIDtoEAJyW7qeibwaM+x0UIE2rCw7lFpm
/jA3xZ09IrdlAiEArvZW6stoHuz15nlgX+axVYLvWPCwR+TD70OH8ChDAlUCIQDi
LpanU8DwkkGU2KO/5jeDgUtA6jUICNkvPf5Ri5Yqag==
-----END RSA PRIVATE KEY-----
```

**(2).生成512位的rsa私钥，输出到指定的文件genrsa.txt。**

```
[root@xuexi tmp]# openssl genrsa -out genrsa.txt 512

[root@xuexi tmp]# cat genrsa.txt
-----BEGIN RSA PRIVATE KEY-----
MIIBOgIBAAJBALKdL1RuPeEVlptzzbBYQM6ItPUXwQtVgPYpJH4cHX6UcoIlNaCt
zV+4fUAjH8ZZxdThsnuCJ9BQRgezTsFv4mUCAwEAAQJAd9le89FRNiItP7vxrb1a
Jvu2KKs6vmcuNH6g3PnylIbaI62vKavTVwcq/5VGqLPXwJWeaQVvLefHAEMnOVLF
oQIhAN7q7QB1JMOGBARnVcM9by/0XtJxkHgmZjwSzLEkJNsdAiEAzR8TodfzT0i/
NmnaykyUWtszKtdcsJfbMFZl872XOekCIQDZwfwK+mQTbBL4iklJE/ZNjhYi1TUf
acNs46B5Wql2MQIgMFzPaC1edKcWTmIO7/u2TuW33rYAaLKlP3RffWSKL2ECIHKF
7avQDhh84y+kdVf4CqJ+DE/N+SZcqZi/lMDs41BG
-----END RSA PRIVATE KEY-----
```

**(3).加密私钥文件，加密的密码为123456。**

```
[root@xuexi tmp]# openssl genrsa -out genrsa.txt -des3 -passout pass:123456 512

[root@xuexi tmp]# cat genrsa.txt
-----BEGIN RSA PRIVATE KEY-----
Proc-Type: 4,ENCRYPTED
DEK-Info: DES-EDE3-CBC,8775D04B98C9C9EE

zx8ehH7qey+588/UVCKu37NFzMvYCBJr9jrryhg7AJtU9eKxFlDIAVNVvNqgG8k+
a5H4a7TCmHB15coFJH52tpmqdGdfo5+758QEgax/YLTPrlBufatb7ZOOpexvJMUe
oenkdYmgzUOj6WRrBUapzwwdhukXu3fpqxmke5ARBSgzsL7amvTLhSt+8ZmhITj6
Rjr55T9rFvCUUCCMKLVYzHgfrTrMTxidv4VsprO1TQ+W//fJOqFqIGJZfbqVLXsb
/0w1vsmzgB0o0k1KHZVj7L4kTGd+j/2nUtc2eaaZUkUdWTu3OflXObg0JVW2a1pG
e67UfEQ0/xhDzFIz1yDyg0v7wozKOSh9c2etsn+NOJHNa7IBsAWX/7Ufz0uxCBuX
0PkJynaFEU6YMoN5RDdDAjnlcwWHqoEbd2RhyYgja+0=
-----END RSA PRIVATE KEY-----
```

其实一般情况下能用到的选项也就`-out`和`numbits`

<a name="openssl_rsa_pkey"></a>

## openssl rsa和openssl pkey

openssl rsa和openssl pkey分别是RSA密钥的处理工具和通用非对称密钥处理工具，它们用法基本一致，所以只举例说明openssl rsa。

它们的用法很简单，基本上就是输入和输出私钥或公钥的作用。

```
openssl rsa [-in filename] [-passin arg] [-passout arg] 
            [-out filename] [-des|-des3|-idea] [-text] 
            [-noout] [-pubin] [-pubout] [-check]

openssl pkey [-passin arg] [-passout arg] [-in filename]
             [-out filename] [-cipher] [-text] [-noout] 
             [-pubin] [-pubout]
 
【openssl rsa选项说明：】
-in filename：指定密钥输入文件。默认读取的是私钥，若指定
             "-pubin"选项将表示读取公钥。将从该文件读取密钥，
             不指定时将从stdin读取。
-out filename：指定密钥输出文件。默认输出私钥，若指定"-pubin"
             或"-pubout"选项都将输出公钥。不指定将输出到stdout。
-pubin：指定该选项时，将显式表明从"-in filename"的filename中读
        取公钥，所以filename必须为公钥文件。不指定该选项时，默认
        是从filename中读取私钥。公钥文件可以通过文件中的公钥标识符
        "---BEGIN PUBLIC KEY---"和"---END PUBLIC KEY---"来辨别。
-pubout：指定该选项时，将显示表明从"-in filename"的filename中提
         取公钥并输出，所以filename文件必须是私钥文件。不指定该选项
         时，默认输出私钥。当设置了"-pubin"时，默认也设置了"-pubout"。
         私钥文件可以通过文件中的私钥标识符"---BEGIN PRIVATE KEY---"
         和"---END PRIVATE KEY---"来辨别。
-noout ：控制不输出任何密钥信息。
-text  ：转换输入和输出的密钥文件格式为纯文本格式。
-check ：检查RSA密钥是否完整未被修改过，只能检测私钥，因为公钥来源于
         私钥。因此选项"-in filename"的filename文件只能是私钥文件。
-des|-des3|-idea：加密输出文件，使得每次读取输出文件时都需要提供密码。
-passin arg ：传递解密密钥文件的密码。密码格式见openssl密码格式。
-passout arg：指定加密输出文件的密码。

【openssl pkey选项说明：】
-cipher：等价于openssl rsa的"-des|-des3|-idea"，例如"-cipher des3"。
```

例如：

**(1).创建一个rsa私钥文件genrsa.pri，然后从中提取rsa公钥到rsa.pub文件中。**

```
[root@xuexi tmp]# openssl genrsa -out genrsa.pri

[root@xuexi tmp]# openssl rsa -in genrsa.pri -pubout -out rsa.pub

[root@xuexi tmp]# cat rsa.pub
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCxitfLsvV58ogZr4hwdsEp7Mne
hShLasCZsE10jGxQLsxoJ7FSBsrnZPB4GBSuwEEXazZdo7547QdLxe9vVL3+YSu3
kPkd30tCTjf2HVywvj3ou2zgEdAIQTfQ0sODV6daO1bKZJxT1fIonEGuIQaefkqR
TxTQOxLIcfR0gayvgQIDAQAB
-----END PUBLIC KEY-----
```

**(2).创建一个加密的rsa私钥文件genrsaK.pri，然后从此文件输出公钥至文件rsaK.pub。**

```
[root@xuexi tmp]# openssl genrsa -out genrsaK.pri -des3 -passout pass:123456
```

此时将提示输入密码才能读取该私钥文件。

```
[root@xuexi tmp]# openssl rsa -in genrsaK.pri -pubout -out rsaK.pub
Enter pass phrase for genrsaK.pri:
```

可以使用"-passin"传递解密的密码。

```
[root@xuexi tmp]# openssl rsa -in genrsaK.pri -pubout -out rsaK.pub -passin pass:123456
```

**(3).移除私钥文件或公钥文件的密码。只需直接输出到新文件即可。以已加密的私钥文件genrsaK.pri为例。**

```
[root@xuexi tmp]# openssl rsa -in genrsaK.pri -out genrsaNK.pri
```

**(4).check检测私钥文件的一致性，查看私钥文件被修改过。**

```
[root@xuexi tmp]# openssl rsa -in genrsaK.pri -check
Enter pass phrase for genrsaK.pri:
RSA key ok
writing RSA key
-----BEGIN RSA PRIVATE KEY-----
MIICXAIBAAKBgQDAXX2ZwqCtcJXiR9xr7lJhslBybeAo07Q1jYK9IT0atbj72jj+
3Eh5vbAjF0R5GF+luTBpGdhTVlt774oj+5m6zvkx785YpW9gRGroN9eglgvUu8iA
9nY30ulIOTEmpi/TfSBVIBL+XbqZ2pLtr05t59RsLkBcqD7huLq28TODTwIDAQAB
AoGALHWvMNl933g0/B6VwFBNtAzNcRUaCPWdIf955xKGl+TGQ1dVcvoguhpwWjvn
dIGAocHigXgaunAsJsHfUJ+3EMJn7SeI25NDraSRdgH6XFK7yKg3ed5Oh4zLmZEx
kEzh91jBkAwwM29/Vv0kbBiV6ZHH/zOxqkCylEaREQou4VECQQDjK6CrYtak+LR+
LbALGSCWeugMU3h2vweNDmTBVwhbHJXe0inCxrBgK9AXSMstnfmcwH1GR/b1Hzdj
U7TnImxpAkEA2McaDnRprd0pExHKeslyX93M/vUXRikr7H8pFke2lLmx8/HGVZHx
erJj8V8sLIIGRc/j0wPoY6hpf67YEeBa9wJBANEPHVWcKBy6JKDaOuB7x1m00khF
qN7e/nv5ew/SoIX40JO2pWfyoe5fY6mJ/DGG6GgxXRiIseTzTW3DYwAy1cECQFGz
WIKyJVI91Ek3n1R/r/eppKVCwi7TPZa4pkebZ5jOE9+Y8+M0SgqwSTKjaAauSqbt
HzRceK12v6w7vXufTykCQDoejWxVGsaxOW8H5D0+8dcF0JVW5hWljcBuvDd0aEJ0
R4tGO1twXzO1dWBbIFJHHIot6W+V1g5CxVPS4QSsdXU=
-----END RSA PRIVATE KEY-----
```

现在随便修改下私钥文件，再检测。

```
[root@xuexi tmp]# openssl rsa -in genrsaK.pri -check
unable to load Private Key
139890935146400:error:0906D066:PEM routines:PEM_read_bio:bad end line:pem_lib.c:802:
```

一般来说，openssl rsa的常用选项就只有`-in filename`、`-out filename`、`-pubout`。

## openssl speed

测试加密算法的性能。

支持的算法有：

```
openssl speed [md2] [mdc2] [md5] [hmac] [sha1] [rmd160] [idea-cbc] 
              [rc2-cbc] [rc5-cbc] [bf-cbc] [des-cbc] [des-ede3] 
              [rc4] [rsa512] [rsa1024] [rsa2048] [rsa4096] [dsa512]
              [dsa1024] [dsa2048] [idea] [rc2] [des] [rsa] [blowfish]
```

测试速度好几秒一个指标，很慢。如果不指定参数将测试所有支持的算法，所以会花很久时间，我的虚拟机上花了十多分钟才测试完所有的算法性能。

例如测试下，dsa512、rsa512和rsa2048加密速度如何。

```
[root@xuexi tmp]# openssl speed dsa512 rsa512 rsa2048
Doing 512 bit private rsa's for 10s: 107496 512 bit private RSA's in 9.99s
Doing 512 bit public rsa's for 10s: 1425095 512 bit public RSA's in 10.00s
Doing 2048 bit private rsa's for 10s: 4623 2048 bit private RSA's in 9.99s
Doing 2048 bit public rsa's for 10s: 153395 2048 bit public RSA's in 9.99s
Doing 512 bit sign dsa's for 10s: 102089 512 bit DSA signs in 10.00s
Doing 512 bit verify dsa's for 10s: 121654 512 bit DSA verify in 9.99s
                  sign    verify    sign/s verify/s
rsa  512 bits 0.000093s 0.000007s  10760.4 142509.5
rsa 2048 bits 0.002161s 0.000065s    462.8  15354.9
                  sign    verify    sign/s verify/s
dsa  512 bits 0.000098s 0.000082s  10208.9  12177.6
```

在10秒时间内，rsa512的私钥处理107496单位，而rsa2048仅处理4623单位，慢了20多倍。

再看签名性能，dsa算法只支持签名不支持加密，而rsa支持加密也支持签名。从上面的结果中可以看到rsa512的签名速度为每秒10760.4，而dsa512的速度为10208.9，速度相差不大。

## openssl rand

生成伪随机数。

```
openssl rand [-out file] [-rand file(s)] [-base64] [-hex] num

选项说明：
-out：指定随机数输出文件，否则输出到标准输出。
-rand file：指定随机数种子文件。种子文件中的字符越随机，
            openssl rand生成随机数的速度越快，随机度越高。
-base64：指定生成的随机数的编码格式为base64。
-hex：指定生成的随机数的编码格式为hex。
num：指定随机数的长度。
```

示例：

```
[root@xuexi tmp]# openssl rand -base64 30;openssl rand -hex 30;openssl rand 30
7Nsce908GnQ5vpDH7Ww44f5ayZxCI0C9MzxSuaLj
af3efb867af2ffa7b9fd2ebd8caa977fc8d26f8a01de86d7b3de8630a25e
9=@oxW[root@xuexi tmp]#  
```

可以看到，不指定`-base64`或`-hex`时生成的随机数是乱码随机数（其实是2进制），且没有`\n`符号。

## openssl passwd

该伪命令用于生成加密的密码。

```
[root@xuexi tmp]# whatis passwd
passwd               (1)  - update user's authentication tokens
passwd               (5)  - password file
passwd [sslpasswd]   (1ssl)  - compute password hashes
```

直接`man passwd`会得到修改用户密码的passwd命令帮助，而不是`openssl passwd`的帮助，所以`man sslpasswd`或者`man openssl-passwd`。

```
[root@xuexi tmp]# man sslpasswd

NAME
       passwd - compute password hashes

SYNOPSIS
       openssl passwd [-crypt] [-1] [-apr1] [-salt string]
       [-in file] [-stdin] [-quiet] {password}
```

使用openssl passwd支持3种加密算法方式：不指定算法时，默认使用`-crypt`。

```
选项说明：

-crypt：UNIX标准加密算法，此为默认算法。如果加盐(-salt)算密码，
        只取盐的前2位，2位后面的所有字符都忽略。
-1(数字)：使用MD5的算法。
-5: 使用SHA256的算法。
-6：使用SHA512的算法。
-apr1(数字)：apache中使用的备选md5算法，不能和"-1"选项一起使用，
            因为apr1本身就默认了md5。htpasswd工具生成的身份验证
            密码就是此方法。
-salt：加密时加点盐，可以增加算法的复杂度。但加了盐会有副作用：盐
       相同，密码相同，加密的结果将一样。
-in file：从文件中读取要计算的密码列表
-stdin：从标准输入中获取要输入的密码
-quiet：生成密码过程中不输出任何信息
```

在命令行中直接输入要加密的密码password或者使用`-salt`时，将不需要交互确认，否则会交互确认密码。

```
[root@xuexi ~]# openssl passwd 123456 ; openssl passwd 123456 
R7J9OiPEN5xUw
C1lvfmeMltEWw
```

由上面的测试可知，使用默认的`-crypt`加密的密码是随机的。但是加入盐后，如果密码一样，盐一样，那么加密结果一样。

```
[root@xuexi ~]# openssl passwd -salt 'xxx' 123456 ; openssl passwd -salt 'xxx' 123456
xxkVQ7YXT9yoE
xxkVQ7YXT9yoE
```

同时也看到了`-crypt`加密算法只取盐的前两位。

如果盐的前两位和密码任意一个不一样，加密结果都不一样。

```
[root@xuexi ~]# openssl passwd -salt 'xyx' 123456;openssl passwd -salt 'xxx' 123456
xyJkVhXGAZ8tM
xxkVQ7YXT9yoE
```

注意，默认的`-crypt`只取盐的前两位字符，所以只要盐的前两位一样，即使第三位不同，结果也是一样的。

```
[root@xuexi ~]# openssl passwd -salt 'xyz' 123456 ; openssl passwd -salt 'xyy' 123456
xyJkVhXGAZ8tM
xyJkVhXGAZ8tM
```

测试下MD5格式的加密算法。

```
[root@xuexi ~]# openssl passwd -1 123456 ; openssl passwd -1 123456     
$1$CJ1eA7bT$4VAJoS3hU/gRTrSQ8r8UQ.
$1$l1uIsNoH$A35cHQ6oGm29IJOas5v7w0
```

可见，结果比`-crypt`的算法更长了，且不加盐时，密码生成是随机的。

```
[root@xuexi ~]# openssl passwd -1 -salt 'abcdefg' 123456 ; openssl passwd -1 -salt 'abcdefg' 123456
$1$abcdefg$a3UbImglR4PCA3x7OvwMX.
$1$abcdefg$a3UbImglR4PCA3x7OvwMX.
```

可以看出，加了盐虽然复杂度增加了，但是也受到了【盐相同，密码相同，则加密结果相同】的限制。另外，盐的长度也不再限于2位了。

再为apache或nginx生成访问网页时身份验证的密码，即basic authentication验证方式的密码。

```
[root@xuexi ~]# openssl passwd -apr1  123456 ; openssl passwd -apr1 123456
$apr1$ydbBroeI$/9YsZR.tJI/GS0YswkQLJ.
$apr1$ncebpB6C$4fnRmlrnL2LPKxrZxCZzJ1
[root@xuexi ~]# openssl passwd -apr1 -salt 'abcdefg' 123456 ;  openssl passwd -apr1 -salt 'abcdefg' 123456
$apr1$abcdefg$PCGBZd8XFTLOgZzLLU3K00
$apr1$abcdefg$PCGBZd8XFTLOgZzLLU3K00
```

同样，加了盐就受到【盐相同，密码相同则加密结果相同】的限制。

openssl passwd生成的密码可以直接用于/etc/shadow文件中，但有些老版本的openssl不支持sha256/sha512，此时可以使用grub-crypt生成，它是一个python脚本，只不过很不幸CentOS 7只有grub2，grub-crypt命令已经没有了。

```
[root@xuexi ~]# grub-crypt --sha-512
Password:
Retype password:
$6$2RCBJT7rELpfX4.Q$iKM5vNShNqUcCiez.JDBgbRkj007eXVVs790UwiOw1PMvB/s/vE7DhyDe8YJ6T8aEtP0Vev5kMReL/nILwLZX/
```

 可以使用语句简单地代替grub-crypt。

```
python -c 'import crypt,getpass;pw=getpass.getpass();print(crypt.crypt(pw) if (pw==getpass.getpass("Confirm: ")) else exit())'
```

grub-crypt和上述python语句都是交互式的。如果要非交互式，稍稍修改下python语句：

```
python -c 'import crypt,getpass;pw="123456";print(crypt.crypt(pw))'
```

## openssl dgst(生成和验证数字签名)

该伪命令是单向加密工具，用于生成文件的摘要信息，也可以进行数字签名，验证数字签名。

首先要明白的是，**数字签名的过程是计算出数字摘要，然后使用私钥对数字摘要进行签名**，而摘要是使用md5、sha512等算法计算得出的，理解了这一点，openssl dgst命令的用法就完全掌握了。

```
openssl dgst [-md5|-sha1|...] [-hex | -binary] [-out filename]
             [-sign filename] [-passin arg] [-verify filename]
             [-prverify filename] [-signature filename] [file...]

选项说明：
file...：指定待签名的文件。
-hex：以hex格式输出数字摘要。如果不以-hex显示，签名或验证签名时很可能乱码。
-binary：以二进制格式输出数字摘要，或以二进制格式进行数字签名。这是默认格式。
-out filename：指定输出文件，若不指定则输出到标准输出。
-sign filename：使用filename中的私钥对file数字签名。
                签名时绝对不能加-hex等格式的选项，否则验证签名失败，亲测。
-signature filename：指定待验证的签名文件。
-verify filename：使用filename中的公钥验证签名。
-prverify filename：使用filename中的私钥验证签名。
-passin arg：传递解密密码。若验证签名时实用的公钥或私钥文件是被加密过的，则需要传递密码来解密。
```

其中，`-passin arg`传递密码的arg的格式见[openssl密码格式](#blogpasswd)。

支持如下几种单向加密算法，即签名时使用的hash算法。

```
-md4        to use the md4 message digest algorithm
-md5        to use the md5 message digest algorithm
-ripemd160  to use the ripemd160 message digest algorithm
-sha        to use the sha message digest algorithm
-sha1       to use the sha1 message digest algorithm
-sha224     to use the sha224 message digest algorithm
-sha256     to use the sha256 message digest algorithm
-sha384     to use the sha384 message digest algorithm
-sha512     to use the sha512 message digest algorithm
-whirlpool  to use the whirlpool message digest algorithm
```

注意：`openssl dgst -md5`和`openssl md5`的作用是一样的，其他单向加密算法也一样，例如`openssl dgst -sha`等价于`openssl sha`。

例如：

**(1).随机生成一段摘要信息。**

```
[root@toystory ~]# echo 123456 | openssl md5
(stdin)= f447b20a7fcbf53a5d5be013ea0b15af
```

**(2).对/tmp/a.txt文件生成MD5和sha512摘要信息。**

```
[root@xuexi tmp]# openssl dgst -md5 a.txt
MD5(a.txt)= 0bbee18df3acef3f0f8496eb7e1d03ad
[root@xuexi tmp]# openssl sha512 a.txt   
SHA512(a.txt)= 1a47bd35ac33100904604e5bd0fb4ebf48b5a1a3c15a5173f17f4affe180d24e1afebbd4f08e08b80ded59383a319c85f978861505e898748b4bef6f07f64e22
```

**(3).生成一个私钥genrsa.pri，然后使用该私钥对/tmp/a.txt文件签名。使用-hex选项，否则默认输出格式为二进制会乱码。**

```
[root@xuexi tmp]# openssl genrsa -out genrsa.pri

[root@xuexi tmp]# openssl dgst -md5 -hex -sign genrsa.pri a.txt
RSA-MD5(a.txt)= 7a6930b06dc6980d1a1fee872df5c8c9c887633c8e2f8b951d40aff4e934b206423914129f66651344859981e33c448f3a61274bded973b387065e9c7909bfcfc1d844e35af1453cc248d58170eb27e948a8de862f21a2b7ee34f512b3cc3cb44537e26c62a409e211320b87f74a8fa5ec1bcc790a7c13ffaa9df9aa8c5ddb64
```

如果要验证签名，那么这个生成的签名要保存到一个文件中，且**一定不能使用"-hex"选项，否则验证签名失败**。以下分别生成使用和不使用hex格式的签名文件以待验证签名测试。

```
[root@xuexi tmp]# openssl dgst -md5 -hex -out md5_hex.sign -sign genrsa.pri a.txt
[root@xuexi tmp]# openssl dgst -md5 -out md5_nohex.sign -sign genrsa.pri a.txt
```

**(4).验证签名。验证签名的过程实际上是对待验证文件新生成签名，然后与已有签名文件进行比对，如果比对结果相同，则验证通过。所以，在验证签名时不仅要给定待验证的签名文件，也要给定相同的算法，相同的私钥或公钥文件以及待签名文件以生成新签名信息。**

以下先测试以私钥来验证数字签名文件。

首先对未使用hex格式的签名文件md5_nohex.sign进行验证。由于生成md5_nohex.sign时使用的是md5算法，所以这里必须也要指定md5算法。

```
[root@xuexi tmp]# openssl dgst -md5 -prverify genrsa.pri -signature md5_nohex.sign a.txt
Verified OK
```

再对使用了hex格式的签名文件md5\_hex.sign进行验证，不论在验证时是否使用了hex选项，结果都是验证失败。

```
[root@xuexi tmp]# openssl dgst -md5 -prverify genrsa.pri -signature md5_hex.sign a.txt  
Verification Failure

[root@xuexi tmp]# openssl dgst -md5 -hex -prverify genrsa.pri -signature md5_hex.sign a.txt
Verification Failure
```

再测试使用公钥来验证数字签名。

```
[root@xuexi tmp]# openssl rsa -in genrsa.pri -pubout -out rsa.pub

[root@xuexi tmp]# openssl dgst -md5 -verify rsa.pub -signature md5_nohex.sign a.txt
Verified OK
```

## openssl rsautl和openssl pkeyutl(非对称加密)

rsautl是rsa的工具，相当于rsa、dgst的部分功能集合，可用于**生成数字签名、验证数字签名、加密和解密文件**。

pkeyutl是非对称加密的通用工具，大体上和rsautl的用法差不多，所以此处只解释rsautl。

```
openssl rsautl [-in file] [-out file] [-inkey file] [-pubin]
               [-certin] [-passin arg] [-sign] [-verify]
               [-encrypt] [-decrypt] [-hexdump]

openssl pkeyutl [-in file] [-out file] [-sigfile file]
                [-inkey file] [-passin arg] [-pubin] [-certin]
                [-sign] [-verify] [-encrypt] [-decrypt] [-hexdump]

共同的选项说明：
-in file：指定输入文件
-out file：指定输出文件
-inkey file：指定密钥输入文件，默认是私钥文件，指定了"-pubin"则表示为
             公钥文件，使用"-certin"则表示为包含公钥的证书文件
-pubin：指定"-inkey file"的file是公钥文件
-certin：使用该选项时，表示"-inkey file"的file是包含公钥的证书文件
-passin arg：传递解密密码。若验证签名时实用的公钥或私钥文件是被加密过
             的，则需要传递密码来解密。

【功能选项：】
-sign：签名并输出签名结果，注意，该选项需要提供RSA私钥文件
-verify：使用验证签名文件
-encrypt：使用公钥加密文件
-decrypt：使用私钥解密文件
【输出格式选项：】
-hexdump：以hex方式输出


openssl pkeyutl选项说明：
sigfile file：待验证的签名文件
```

其中，`-passin arg`传递密码的arg的格式见[openssl密码格式](#blogpasswd)。

rsautl命令的用法和rsa、dgst不太一样。首先，它的前提是已经有非对称密钥，所有的命令操作都用到公钥或私钥来处理；再者，该命令使用`-in`选项来指定输入文件，而不像dgst一样可以把输入文件放在命令的结尾；最后，该命令使用的密钥文件、签名文件、证书文件都通过`-inkey`选项指定，再通过各功能的选项搭配来实现对应的功能。

**注意rsautl和pkeyutl的缺陷是默认只能对短小的文件进行操作，否则将报类似如下的错误信息。**

```
140341340976968:error:0406C06E:rsa routines:RSA_padding_add_PKCS1_type_1:data too large for key size:rsa_pk1.c:73:
```

因为这两个工具签名和验证签名的功能和openssl dgst命令差不多，且自身又有缺陷，所以就不举例说明。此处仅给出对短小文件的非对称加密和解密示例。

**(1).使用公钥加密b.txt文件，注意待加密文件b.txt必须是短小文件，且不建议使用-hexdump输出，否则解密时可能超出文件的长度。**

```
[root@xuexi tmp]# openssl genrsa -out genrsa.pri   # 生成私钥

[root@xuexi tmp]# openssl rsa -in genrsa.pri -pubout -out rsa.pub   # 从私钥中提取公钥

[root@xuexi tmp]# openssl rsautl -encrypt -in b.txt -out b_crypt.txt -inkey rsa.pub -pubin
```

查看非对称加密后的文件b\_crypt.txt。

```
[root@xuexi tmp]# cat b_crypt.txt
H[1]=p?I,:=)Iڪ;Yx٩,vbot@xuexi tmp]#
```

**(2).使用私钥解密b\_crypt.txt文件。**

```
[root@xuexi tmp]# openssl rsautl -decrypt -in b_crypt.txt -out b_decrypt.txt -inkey genrsa.pri

[root@xuexi tmp]# cat b_decrypt.txt
UUID=d505113c-daa6-4c17-8b03-b3551ced2305 swap swap defaults  0 0
```

## openssl enc(对称加密)

对称加密工具。了解对称加密的原理后就很简单了，原理部分见下文。

```
openssl enc -ciphername [-in filename] [-out filename] [-pass arg]
                        [-e] [-d] [-a/-base64] [-k password]
                        [-S salt] [-salt] [-md] [-p/-P]

选项说明：
-ciphername：指定对称加密算法(如des3)，可独立于enc直接使用，
             如openssl des3或openssl enc -des3。推荐在enc后
             使用，这样不依赖于硬件
-in filename ：输入文件，不指定时默认是stdin
-out filename：输出文件，不指定时默认是stdout
-e：对输入文件加密操作，不指定时默认就是该选项
-d：对输入文件解密操作，只有显示指定该选项才是解密
-pass：传递加、解密时的明文密码。若验证签名时实用的公钥或私钥文件
       是被加密过的，则需要传递密码来解密。
-k     ：已被"-pass"替代，现在还保留是为了兼容老版本的openssl
-base64：在加密后和解密前进行base64编码或解密，不指定时默认是二进制。
         注意，编码不是加解密的一部分，而是加解密前后对数据的格式"整理"
-a：等价于-base64
-salt：单向加密时使用salt复杂化单向加密的结果，此为默认选项，且使用随
       机salt值
-S salt：不使用随机salt值，而是自定义salt值，但只能是16进制范围内字符
         的组合，即"0-9a-fA-F"的任意一个或多个组合
-p：打印加解密时salt值、key值和IV初始化向量值（也是复杂化加密的一种方
    式），解密时还输出解密结果，见后文示例
-P：和-p选项作用相同，但是打印时直接退出工具，不进行加密或解密操作
-md：指定单向加密算法，默认md5。该算法是拿来加密key部分的，见后文分析。
```

其中，`-pass arg`传递密码的arg的格式见[openssl密码格式](#blogpasswd)。

支持的单向加密算法有：

```
-md4       to use the md4 message digest algorithm
-md5       to use the md5 message digest algorithm
-ripemd160 to use the ripemd160 message digest algorithm
-sha       to use the sha message digest algorithm
-sha1      to use the sha1 message digest algorithm
-sha224    to use the sha224 message digest algorithm
-sha256    to use the sha256 message digest algorithm
-sha384    to use the sha384 message digest algorithm
-sha512    to use the sha512 message digest algorithm
-whirlpool to use the whirlpool message digest algorithm
```

支持的对称加密算法有：

```
-aes-128-cbc               -aes-128-cbc-hmac-sha1     -aes-128-cfb             
-aes-128-cfb1              -aes-128-cfb8              -aes-128-ctr             
-aes-128-ecb               -aes-128-gcm               -aes-128-ofb             
-aes-128-xts               -aes-192-cbc               -aes-192-cfb             
-aes-192-cfb1              -aes-192-cfb8              -aes-192-ctr             
-aes-192-ecb               -aes-192-gcm               -aes-192-ofb             
-aes-256-cbc               -aes-256-cbc-hmac-sha1     -aes-256-cfb             
-aes-256-cfb1              -aes-256-cfb8              -aes-256-ctr             
-aes-256-ecb               -aes-256-gcm               -aes-256-ofb             
-aes-256-xts               -aes128                    -aes192                  
-aes256                    -bf                        -bf-cbc                  
-bf-cfb                    -bf-ecb                    -bf-ofb                  
-blowfish                  -camellia-128-cbc          -camellia-128-cfb        
-camellia-128-cfb1         -camellia-128-cfb8         -camellia-128-ecb        
-camellia-128-ofb          -camellia-192-cbc          -camellia-192-cfb        
-camellia-192-cfb1         -camellia-192-cfb8         -camellia-192-ecb        
-camellia-192-ofb          -camellia-256-cbc          -camellia-256-cfb        
-camellia-256-cfb1         -camellia-256-cfb8         -camellia-256-ecb        
-camellia-256-ofb          -camellia128               -camellia192             
-camellia256               -cast                      -cast-cbc                
-cast5-cbc                 -cast5-cfb                 -cast5-ecb               
-cast5-ofb                 -des                       -des-cbc                 
-des-cfb                   -des-cfb1                  -des-cfb8                
-des-ecb                   -des-ede                   -des-ede-cbc             
-des-ede-cfb               -des-ede-ofb               -des-ede3                
-des-ede3-cbc              -des-ede3-cfb              -des-ede3-cfb1           
-des-ede3-cfb8             -des-ede3-ofb              -des-ofb                
-des3                      -desx                      -desx-cbc                
-id-aes128-GCM             -id-aes128-wrap            -id-aes128-wrap-pad      
-id-aes192-GCM             -id-aes192-wrap            -id-aes192-wrap-pad      
-id-aes256-GCM             -id-aes256-wrap            -id-aes256-wrap-pad      
-id-smime-alg-CMS3DESwrap  -idea                      -idea-cbc                 
-idea-cfb                  -idea-ecb                  -idea-ofb                
-rc2                       -rc2-40-cbc                -rc2-64-cbc              
-rc2-cbc                   -rc2-cfb                   -rc2-ecb                 
-rc2-ofb                   -rc4                       -rc4-40                  
-rc4-hmac-md5              -seed                      -seed-cbc                
-seed-cfb                  -seed-ecb                  -seed-ofb
```

在给出`openssl enc`命令用法示例之前，先解释下对称加密和解密的原理和过程。

对称加解密时，它们使用的密码是完全相同的，例如"123456"，但这是密码，且是明文密码，非常不安全，所以应该对此简单密码进行复杂化。最直接的方法是使用单向加密计算出明文密码的hash值，单向加密后新生成的密码已经比较安全(称之为密钥比较好)，可以作为对称加密时的对称密钥。另外，由于同一单向加密算法对相同明文密码的计算结果是完全一致的，这样解密时使用相同的单向加密算法就能计算出完全相同的密钥，也就是解密时的对称密钥。如果想要更安全，还可以在对称加密后对加密文件进行重新编码，如使用"base64"、二进制或hex编码方式进行编码，但对应的在解密前就需要先解码，解码后才能解密。

所以，将对称加、解密的机制简单概括如下：

- 对称加密机制：根据指定的单向加密算法，对输入的明文密码进行单向加密(默认是md5)，得到固定长度的加密密钥，即对称密钥，再根据指定的对称加密算法，使用对称密钥加密文件，最后重新编码加密后的文件。**即单向加密明文密码结果作为对称密钥、使用对称密钥加密文件、对文件重新编码。**

- 对称解密机制：**先解码文件，再根据单向加密算法对解密时输入的明文密码计算得到对称密钥，依此对称密钥对称解密解码后的文件。**

**因此，解密过程中使用的解码方式、单向加密和对称加密算法都必须一致，且输入的密码必须是正确密码。**但需要注意的一点是，解密时可以不指定salt，因为加密时使用的salt会记录下来，解密时可以读取该salt。

如下图所示，分别是加密和解密过程示意图。

![](E:\onedrive\docs\blog_imgs\733013-20170703154045815-1275400373.png)

![](E:\onedrive\docs\blog_imgs\733013-20170703154059315-740270087.png)

示例：

以加密/etc/fstab的备份文件/tmp/test.txt为例。

**(1).首先测试openssl enc的编码功能。由于未指定密码选项"-k"或"-pass"，所以仅仅只进行编码而不进行加密，因此也不会提示输入密码。**

```
[root@xuexi tmp]# openssl enc -a -in test.txt -out test_base64.txt

[root@xuexi tmp]# cat test_base64.txt
CiMKIyAvZXRjL2ZzdGFiCiMgQ3JlYXRlZCBieSBhbmFjb25kYSBvbiBUaHUgTWF5
IDExIDA0OjE3OjQ0IDIwMTcKIwojIEFjY2Vzc2libGUgZmlsZXN5c3RlbXMsIGJ5
IHJlZmVyZW5jZSwgYXJlIG1haW50YWluZWQgdW5kZXIgJy9kZXYvZGlzaycKIyBT
ZWUgbWFuIHBhZ2VzIGZzdGFiKDUpLCBmaW5kZnMoOCksIG1vdW50KDgpIGFuZC9v
ciBibGtpZCg4KSBmb3IgbW9yZSBpbmZvCiMKVVVJRD1iMmE3MGZhZi1hZWE0LTRk
OGUtOGJlOC1jNzEwOWFjOWM4YjggLyAgICAgICAgICAgICAgICAgICAgICAgeGZz
ICAgICBkZWZhdWx0cyAgICAgICAgMCAwClVVSUQ9MzY3ZDZhNzctMDMzYi00MDM3
LWJiY2ItNDE2NzA1ZWFkMDk1IC9ib290ICAgICAgICAgICAgICAgICAgIHhmcyAg
ICAgZGVmYXVsdHMgICAgICAgIDAgMApVVUlEPWQ1MDUxMTNjLWRhYTYtNGMxNy04
YjAzLWIzNTUxY2VkMjMwNSBzd2FwICAgICAgICAgICAgICAgICAgICBzd2FwICAg
IGRlZmF1bHRzICAgICAgICAwIDAK
```

再以base64格式进行解码。

```
[root@xuexi tmp]# openssl enc -a -d -in test_base64.txt              
 
#
# /etc/fstab
# Created by anaconda on Thu May 11 04:17:44 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=<UUID_VALUE> /        xfs  defaults  0 0
UUID=<UUID_VALUE> /boot    xfs  defaults  0 0
UUID=<UUID_VALUE> swap     swap defaults  0 0
```

实际上，上述编码和解码的过程严格地说也是对称加密和解密，因为`openssl enc`默认会带上加密选项"-e"，只不过因为没有指定输入密码选项，使用的加密密码为空而已，且单向加密算法使用的也是默认值。解密时也一样。

**(2).测试使用des3对称加密算法加密test.txt文件。**

```
[root@xuexi tmp]# openssl enc -a -des3 -in test.txt -out test.1 -pass pass:123456 -md md5
```

加密后，查看加密后文件test.1的结果。

```
[root@xuexi tmp]# cat test.1
U2FsdGVkX1+c/d4NsXnY6Pd7rcZjGSsMRJWQOP0s5sxH6aLE5iCYjKEAbGac//iR
wkUUh6a57OpUA3+OOCKB4z+IxBcKo67BUDGR9vYeCfkobH9F+mSfVzZbXBrJmxwf
921tJ+8K+yKB6DjJfufpW+DWXmH8MFyvK60wnYHsfUQOp81EvaUtEfqEKIS8hgg7
4NTOyww+/VMDdc2wmkf08XNQUPlVtLaSx3vuBisxRdu8raiKWGGOB7qCwELCxDqu
NaRCIh0VjjffGohAOMMsAQ2kFCDUKx0Z4Df5fvifhPXoHfsj2lI216BPG5Cy88K2
KV78DoBm4pnMAymo/HRRF95LjvWYZIN88hIVN67u2j9zqSGeuyJakMyDVhYYmrHl
sMr2YTbTwus2DiO6qAzt/0a9nocTVKfGR81Xsh0a0ZudjtrMl5H36YJawpldvUCa
DzXPsbpQrp0VGi2HvJ4EVKKEx2uh8XYWmJ4ytj1s1wtCR6wQhmERtInGwULWTyI+
agXStSB5XzsvAJRJvexsaNycj5lAoQ8O6YXEj7B0inB7nBQTFbwkXyvJqXpr1179
i67leYc59OvlhRMA+GLW4g/Mg5dN5SBmgt1ChOJs4887zAUyLYrLvR4zDK6IQN/M
P6F15c9V+m9pw2t32sUQQmYrYqOV/AQf0t0EwvA0Myjmfqtvmp555Q==
```

解密文件test.1。

```
[root@xuexi tmp]# openssl enc -a -des3 -d -in test.1 -out test.2 -pass pass:123456 -md md5 

[root@xuexi tmp]# cat test.2
 
#
# /etc/fstab
# Created by anaconda on Thu May 11 04:17:44 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=<UUID_VALUE> /      xfs     defaults   0 0
UUID=<UUID_VALUE> /boot  xfs     defaults   0 0
UUID=<UUID_VALUE> swap   swap    defaults   0 0
```

**(3).加密时带上点盐salt。其实不写时默认就已经加入了，只不过是加入随机盐值。使用-S可以指定明确要使用的盐的值。但是盐的值只能是16进制范围内字符的组合，即`0-9a-fA-F`的任意一个或多个组合。**

```
[root@xuexi tmp]# openssl enc -a -des3 -S 'Fabc' -in test.txt -out test.1 -pass pass:123456 -md md5    
```

解密。解密时不用指定salt值，即使指定了也不会影响解密结果。      

```
[root@xuexi tmp]# openssl enc -a -des3 -d -in test.1 -pass pass:123456 -md md5               
 
#
# /etc/fstab
# Created by anaconda on Thu May 11 04:17:44 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=<UUID_VALUE> /        xfs     defaults   0 0
UUID=<UUID_VALUE> /boot    xfs     defaults   0 0
UUID=<UUID_VALUE> swap     swap    defaults   0 0

[root@xuexi tmp]# openssl enc -a -des3 -d -S 'Fabcxdasd' -in test.1 -pass pass:123456 -md md5
 
#
# /etc/fstab
# Created by anaconda on Thu May 11 04:17:44 2017
#
# Accessible filesystems, by reference, are maintained under '/dev/disk'
# See man pages fstab(5), findfs(8), mount(8) and/or blkid(8) for more info
#
UUID=<UUID_VALUE> /      xfs     defaults   0 0
UUID=<UUID_VALUE> /boot  xfs     defaults   0 0
UUID=<UUID_VALUE> swap   swap    defaults   0 0
```

**(4).在测试下"-p"和"-P"选项的输出功能。小写字母p不仅输出密钥算法结果，还输出加解密的内容，而大写字母P则只输出密钥算法结果。**

加密时的情况。

```
[root@xuexi tmp]# openssl enc -a -des3 -S 'Fabc' -in test.txt -out test.1 -pass pass:123456 -md md5 -p
salt=FABC000000000000
key=885FC58E6C822AEFC8032B4B98FA0355F8482BD654739F3D
iv =5128FDED01EE1499
```

其中key就是单向加密明文密码后得到的对称密钥，iv是密码运算时使用的向量值。

再看解密时的情况，此处加上了salt。

```
[root@xuexi tmp]# openssl enc -a -des3 -d -S 'Fabc' -in test.1 -pass pass:123456 -md md5 -P
salt=FABC000000000000
key=885FC58E6C822AEFC8032B4B98FA0355F8482BD654739F3D
iv =5128FDED01EE1499
```

若解密时不指定salt，或者随意指定salt，结果如下。

```
[root@xuexi tmp]# openssl enc -a -des3 -d -in test.1 -pass pass:123456 -md md5 -P         
salt=FABC000000000000
key=885FC58E6C822AEFC8032B4B98FA0355F8482BD654739F3D
iv =5128FDED01EE1499
[root@xuexi tmp]# openssl enc -a -des3 -S 'FabM' -d -in test.1 -pass pass:123456 -md md5 -P
salt=FABC000000000000
key=885FC58E6C822AEFC8032B4B98FA0355F8482BD654739F3D
iv =5128FDED01EE1499
```

可见，解密时，只要指定和加密时相同编码格式和单向加密算法，密钥的结果就是一样的，且解密时明确指定salt是无意义的，因为它可以读取到加密时使用的salt。

甚至，解密时指定不同的对称加密算法，密钥结果也是一样的。

```
[root@xuexi tmp]# openssl enc -a -desx -d -in test.1 -pass pass:123456 -md md5 -p 
salt=FABC000000000000
key=885FC58E6C822AEFC8032B4B98FA0355F8482BD654739F3D
iv =5128FDED01EE1499
```

由此，能推理出对称加密时使用的对称密钥和对称算法是毫无关系的。

## openssl dhparam(密钥交换)

openssl dhparam用于生成和管理dh文件。dh(Diffie-Hellman)是著名的密钥交换协议，或称为密钥协商协议，它可以保证通信双方安全地交换密钥。但注意，它不是加密算法，所以不提供加密功能，仅仅只是保护密钥交换的过程。在openvpn中就使用了该交换协议。关于dh算法的整个过程，见下文。

openssl dhparam命令集合了老版本的`openssl dh`和`openssl gendh`，后两者可能已经失效了，即使存在也仅表示未来另有用途。

```
openssl dhparam [-in filename] [-out filename] [-dsaparam]
                [-noout] [-text] [-rand file(s)] [numbits]
选项说明：
-in filename：从filename文件中读取密钥交换协议参数。
-out filename：输出密钥交换协议参数到filename文件。
-dsaparam：指定此选项将使用dsa交换协议替代dh交换协议。
           虽然生成速度更快，但更不安全。
-noout：禁止输出任何信息。
-text：以文本格式输出dh协议。
-rand：指定随机数种子文件。
numbits：指定生成的长度。
```

注意，dh协议文件生成速度随长度增长而急剧增长，使用随机数种子可以加快生成速度。

例如：生成1024长度的交换协议文件，其消耗的时间2秒不到。

```sql
[root@xuexi tmp]# time openssl dhparam -out dh.pem 1024               

Generating DH parameters, 1024 bit long safe prime, generator 2
This is going to take a long time

real    0m1.762s
user    0m1.608s
sys     0m0.017s
```

但生成长度2048的交换协议文件用了4分多钟，可见长度增长会导致协议生成的时间急剧增长。

```sql
[root@xuexi ~]# time openssl dhparam -out dh.pem 2048

    .........
    .........
real    4m36.606s
user    4m14.404s
sys     0m0.538s
```

而使用了64位随机数种子的同样命令只需50秒钟。

```sql
[root@xuexi tmp]# time openssl dhparam -rand rand.seed -out dh.pem 2048

    .........
    .........
real    0m50.264s
user    0m46.039s
sys     0m0.104s
```

openssl命令实现的是各种算法和加密功能，它的cpu的使用率会非常高，再结合dhparam，可以使得openssl dhparam作为一个不错的cpu压力测试工具，并且可以长时间飙高cpu使用率。

### DH密钥协商过程

密钥交换协议(DH)的大概过程是这样的(了解即可，可网上搜索完整详细的过程)：

(1).双方协商一个较大的质数并共享，这个质数是种子数。

(2).双方都协商好一个加密生成器(一般是AES)。

(3).双方各自提出另一个质数，这次双方提出的质数是互相保密的。这个质数被认为是私钥(不是非对称加密的私钥)。

(4).双方使用自己的私钥(即各自保密的质数)、加密生成器以及种子数(即共享的质数)派生出一个公钥(由上面的私钥派生而来，不是非对称加密的公钥)。

(5).双方交换派生出的公钥。

(6).接收方使用自己的私钥(各自保密的质数)、种子数(共享的质数)以及接收到的对方公钥计算出共享密钥(session key)。尽管双方的session key是使用对方的公钥以及自己的私钥计算的，但因为使用的算法，能保证双方计算出的session key相同。

(7).这个session key将用于加密后续通信。例如，ssh连接过程中，使用host key对session key进行签名，然后验证指纹来完成主机认证的过程(见<https://www.junmajinlong.com/linux/ssh#host_auth>)。

![](E:\onedrive\docs\blog_imgs\733013-20180909222227055-1193041503.png)

在此可见，在计算session key过程中，双方使用的公钥、私钥是相反的。但因为DH算法的原因，它能保证双方生成的session key是一致的。而且因为双方在整个过程中是完全平等的，没有任何一方能掌控协商的命脉，再者session key没有在网络上进行传输，使得使用session key做对称加密的数据传输是安全的。

