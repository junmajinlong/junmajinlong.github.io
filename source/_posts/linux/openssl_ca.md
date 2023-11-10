---
title: openssl申请证书、颁发证书、自建CA
p: linux/openssl_ca.md
date: 2021-03-15 15:18:18
tags: Linux
categories: Linux
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# openssl申请证书、颁发证书、自建CA

本文介绍openssl与证书相关方面的内容。openssl的其它加密解密、数字签名、编码等用法参考[openssl命令用法详解][1]。

## openssl req(生成证书请求和自建CA)

伪命令req大致有3个功能：生成证书请求文件、验证证书请求文件和创建根CA。由于openssl req命令选项较多，所以先各举几个例子，再集中给出openssl req的选项说明。若已熟悉openssl req和证书请求相关知识，可直接跳至后文[openssl req选项整理](#blogopensslreq)，若不熟悉，建议从前向后一步一步阅读。

首先说明下生成证书请求需要什么：申请者需要将自己的信息及其公钥放入证书请求中。但在实际操作过程中，所需要提供的是私钥而非公钥，因为它会自动从私钥中提取公钥。另外，还需要将提供的数据进行数字签名(使用单向加密)，保证该证书请求文件的完整性和一致性，防止他人盗取后进行篡改，例如黑客将为`www.example.com`所申请的证书请求文件中的公司名改成对方的公司名称，如果能够篡改成功，则签署该证书请求时，所颁发的证书信息中将变成他人信息。

![](E:\onedrive\docs\blog_imgs\733013-20170703235048206-116648006.png)

所以第一步就是先创建出私钥pri\_key.pem。其实私钥文件是非必需的，因为openssl req在需要它的时候会自动创建在特定的路径下，此处为了举例说明，所以创建它。

```
[root@xuexi tmp]# openssl genrsa -out pri_key.pem
```

**(1).根据私钥pri\_key.pem生成一个新的证书请求文件。其中"-new"表示新生成一个新的证书请求文件，"-key"指定私钥文件，"-out"指定输出文件，此处输出文件即为证书请求文件。**

```
[root@xuexi tmp]# openssl req -new -key pri_key.pem -out req1.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:FJ
Locality Name (eg, city) [Default City]:XM
Organization Name (eg, company) [Default Company Ltd]:.
Organizational Unit Name (eg, section) []:.
Common Name (eg, your name or your server's hostname) []:www.youwant.com
Email Address []:
 
# 下面两项通常不考虑，除非你知道在干什么
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:.
An optional company name []:.
```

在敲下回车键后，默认会进入交互模式让你提供你个人的信息，需要注意的是，如果某些信息不想填可以选择使用默认值，也可以选择留空不填，直接回车将选择使用默认值，输入点"."将表示该信息项留空。但某些项是必填项，否则未来证书签署时将失败。如"Common Name"，它表示的是为哪个域名、子域名或哪个主机申请证书，未来证书请求被签署后将只能应用于"Common Name"所指定的地址。具体哪些必填项还需要看所使用的配置文件(默认的配置文件为/etc/pki/tls/openssl.cnf)中的定义，此处暂且不讨论配置相关内容，仅提供Common Name即可。

除了"-new"选项，使用"-newkey"选项也能创建证书请求文件，此处暂不举例说明"-newkey"的用法，后文会有示例。

**(2).查看证书请求文件内容。**

现在已经生成了一个新的证书请求文件req1.csr。查看下该证书请求文件的内容。

```
[root@xuexi tmp]# cat req1.csr
-----BEGIN CERTIFICATE REQUEST-----  # 证书请求的内容
MIIBgDCB6gIBADBBMQswCQYDVQQGEwJDTjELMAkGA1UECAwCRkoxCzAJBgNVBAcM
AlhNMRgwFgYDVQQDDA93d3cueW91d2FudC5jb20wgZ8wDQYJKoZIhvcNAQEBBQAD
gY0AMIGJAoGBAMbx9bfsC0GTn7DijfGFs56Fb8atX9ABRDE/wmE74jXjdfbH4ZOg
Te0Orlu5pA4jqXDgSLzlQvjD6QsyhToyvtyQbgGSfXSVOPcgfAohDNo9t6+mnvs/
5rFQJ1+uI6gsLMbwQBJidLGnM1pOvFo2671Vm2jewDLVweGP5wmIfDyLAgMBAAGg
ADANBgkqhkiG9w0BAQUFAAOBgQAqKYjNKKpNCvwDNeDeYynOx1XD/OYgAU43Sq03
aRUcKenqICkvkXkUE+H0lYMtXcDL/rgDyjlKvwartgZ/ngoKSwtXhd4UivII2hNN
jolE3gfe8KGjMpnX/8oxkJIoSTETqee+11ez8E2fya1DwoQnKpXjTt5qya8VWflt
DG8WmA==
-----END CERTIFICATE REQUEST-----
```

更具体的可以使用`openssl req`命令查看。命令如下，其中"-in"选项指定的是证书请求文件。

```
[root@xuexi tmp]# openssl req -in req1.csr
-----BEGIN CERTIFICATE REQUEST-----             # 证书请求的内容
MIIBgDCB6gIBADBBMQswCQYDVQQGEwJDTjELMAkGA1UECAwCRkoxCzAJBgNVBAcM
AlhNMRgwFgYDVQQDDA93d3cueW91d2FudC5jb20wgZ8wDQYJKoZIhvcNAQEBBQAD
gY0AMIGJAoGBAMbx9bfsC0GTn7DijfGFs56Fb8atX9ABRDE/wmE74jXjdfbH4ZOg
Te0Orlu5pA4jqXDgSLzlQvjD6QsyhToyvtyQbgGSfXSVOPcgfAohDNo9t6+mnvs/
5rFQJ1+uI6gsLMbwQBJidLGnM1pOvFo2671Vm2jewDLVweGP5wmIfDyLAgMBAAGg
ADANBgkqhkiG9w0BAQUFAAOBgQAqKYjNKKpNCvwDNeDeYynOx1XD/OYgAU43Sq03
aRUcKenqICkvkXkUE+H0lYMtXcDL/rgDyjlKvwartgZ/ngoKSwtXhd4UivII2hNN
jolE3gfe8KGjMpnX/8oxkJIoSTETqee+11ez8E2fya1DwoQnKpXjTt5qya8VWflt
DG8WmA==
-----END CERTIFICATE REQUEST-----
```

查看请求文件时，可以结合其他几个选项输出特定的内容。"-text"选项表示以文本格式输出证书请求文件的内容。

```
[root@xuexi tmp]# openssl req -in req1.csr -text
Certificate Request:           # 此为证书请求文件头
    Data:
        Version: 0 (0x0)
        # 此为提供的个人信息，注意左侧标头为"Subject"，这是很重要的一项
        Subject: C=CN, ST=FJ, L=XM, CN=www.youwant.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption   # 使用的公钥算法
                Public-Key: (1024 bit)            # 公钥的长度
                Modulus:
                    00:c6:f1:f5:b7:ec:0b:41:93:9f:b0:e2:8d:f1:85:
                    b3:9e:85:6f:c6:ad:5f:d0:01:44:31:3f:c2:61:3b:
                    e2:35:e3:75:f6:c7:e1:93:a0:4d:ed:0e:ae:5b:b9:
                    a4:0e:23:a9:70:e0:48:bc:e5:42:f8:c3:e9:0b:32:
                    85:3a:32:be:dc:90:6e:01:92:7d:74:95:38:f7:20:
                    7c:0a:21:0c:da:3d:b7:af:a6:9e:fb:3f:e6:b1:50:
                    27:5f:ae:23:a8:2c:2c:c6:f0:40:12:62:74:b1:a7:
                    33:5a:4e:bc:5a:36:eb:bd:55:9b:68:de:c0:32:d5:
                    c1:e1:8f:e7:09:88:7c:3c:8b
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    # 为请求文件数字签名时使用的算法
    Signature Algorithm: sha1WithRSAEncryption
         2a:29:88:cd:28:aa:4d:0a:fc:03:35:e0:de:63:29:ce:c7:55:
         c3:fc:e6:20:01:4e:37:4a:ad:37:69:15:1c:29:e9:ea:20:29:
         2f:91:79:14:13:e1:f4:95:83:2d:5d:c0:cb:fe:b8:03:ca:39:
         4a:bf:06:ab:b6:06:7f:9e:0a:0a:4b:0b:57:85:de:14:8a:f2:
         08:da:13:4d:8e:89:44:de:07:de:f0:a1:a3:32:99:d7:ff:ca:
         31:90:92:28:49:31:13:a9:e7:be:d7:57:b3:f0:4d:9f:c9:ad:
         43:c2:84:27:2a:95:e3:4e:de:6a:c9:af:15:59:f9:6d:0c:6f:
         16:98
-----BEGIN CERTIFICATE REQUEST-----                    
MIIBgDCB6gIBADBBMQswCQYDVQQGEwJDTjELMAkGA1UECAwCRkoxCzAJBgNVBAcM
AlhNMRgwFgYDVQQDDA93d3cueW91d2FudC5jb20wgZ8wDQYJKoZIhvcNAQEBBQAD
gY0AMIGJAoGBAMbx9bfsC0GTn7DijfGFs56Fb8atX9ABRDE/wmE74jXjdfbH4ZOg
Te0Orlu5pA4jqXDgSLzlQvjD6QsyhToyvtyQbgGSfXSVOPcgfAohDNo9t6+mnvs/
5rFQJ1+uI6gsLMbwQBJidLGnM1pOvFo2671Vm2jewDLVweGP5wmIfDyLAgMBAAGg
ADANBgkqhkiG9w0BAQUFAAOBgQAqKYjNKKpNCvwDNeDeYynOx1XD/OYgAU43Sq03
aRUcKenqICkvkXkUE+H0lYMtXcDL/rgDyjlKvwartgZ/ngoKSwtXhd4UivII2hNN
jolE3gfe8KGjMpnX/8oxkJIoSTETqee+11ez8E2fya1DwoQnKpXjTt5qya8VWflt
DG8WmA==
-----END CERTIFICATE REQUEST-----
```

将"-text"和"-noout"结合使用，则只输出证书请求的文件头部分。

```
[root@xuexi tmp]# openssl req -in req1.csr -noout -text
Certificate Request:
    Data:
        Version: 0 (0x0)
        Subject: C=CN, ST=FJ, L=XM, CN=www.youwant.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (1024 bit)
                Modulus:
                    00:c6:f1:f5:b7:ec:0b:41:93:9f:b0:e2:8d:f1:85:
                    b3:9e:85:6f:c6:ad:5f:d0:01:44:31:3f:c2:61:3b:
                    e2:35:e3:75:f6:c7:e1:93:a0:4d:ed:0e:ae:5b:b9:
                    a4:0e:23:a9:70:e0:48:bc:e5:42:f8:c3:e9:0b:32:
                    85:3a:32:be:dc:90:6e:01:92:7d:74:95:38:f7:20:
                    7c:0a:21:0c:da:3d:b7:af:a6:9e:fb:3f:e6:b1:50:
                    27:5f:ae:23:a8:2c:2c:c6:f0:40:12:62:74:b1:a7:
                    33:5a:4e:bc:5a:36:eb:bd:55:9b:68:de:c0:32:d5:
                    c1:e1:8f:e7:09:88:7c:3c:8b
                Exponent: 65537 (0x10001)
        Attributes:
            a0:00
    # 为请求文件数字签名时使用的算法
    Signature Algorithm: sha1WithRSAEncryption
         2a:29:88:cd:28:aa:4d:0a:fc:03:35:e0:de:63:29:ce:c7:55:
         c3:fc:e6:20:01:4e:37:4a:ad:37:69:15:1c:29:e9:ea:20:29:
         2f:91:79:14:13:e1:f4:95:83:2d:5d:c0:cb:fe:b8:03:ca:39:
         4a:bf:06:ab:b6:06:7f:9e:0a:0a:4b:0b:57:85:de:14:8a:f2:
         08:da:13:4d:8e:89:44:de:07:de:f0:a1:a3:32:99:d7:ff:ca:
         31:90:92:28:49:31:13:a9:e7:be:d7:57:b3:f0:4d:9f:c9:ad:
         43:c2:84:27:2a:95:e3:4e:de:6a:c9:af:15:59:f9:6d:0c:6f:
         16:98
```

还可以只输出subject部分的内容。

```
[root@xuexi tmp]# openssl req -in req2.csr -subject -noout
subject=/C=CN/ST=FJ/L=XM/CN=www.youwant.com
```

也可以使用"-pubkey"输出证书请求文件中的公钥内容。如果从申请证书请求时所提供的私钥中提取出公钥，这两段公钥的内容是完全一致的。

```
[root@xuexi tmp]# openssl req -in req1.csr -pubkey -noout
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDG8fW37AtBk5+w4o3xhbOehW/G
rV/QAUQxP8JhO+I143X2x+GToE3tDq5buaQOI6lw4Ei85UL4w+kLMoU6Mr7ckG4B
kn10lTj3IHwKIQzaPbevpp77P+axUCdfriOoLCzG8EASYnSxpzNaTrxaNuu9VZto
3sAy1cHhj+cJiHw8iwIDAQAB
-----END PUBLIC KEY-----
[root@xuexi tmp]# openssl rsa -in pri_key.pem -pubout
writing RSA key
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQDG8fW37AtBk5+w4o3xhbOehW/G
rV/QAUQxP8JhO+I143X2x+GToE3tDq5buaQOI6lw4Ei85UL4w+kLMoU6Mr7ckG4B
kn10lTj3IHwKIQzaPbevpp77P+axUCdfriOoLCzG8EASYnSxpzNaTrxaNuu9VZto
3sAy1cHhj+cJiHw8iwIDAQAB
-----END PUBLIC KEY-----
```

**(3).指定证书请求文件中的签名算法。**

注意到证书请求文件的头部分有一项是"Signature Algorithm"，它表示使用的是哪种数字签名算法。默认使用的是sha1，还支持md5、sha512等，更多可支持的签名算法见`openssl dgst --help`中所列出内容。例如此处指定md5算法。

```
[root@xuexi tmp]# openssl req -new -key pri_key.pem -out req2.csr -md5

[root@xuexi tmp]# openssl req -in req2.csr -noout -text | grep Algo
            Public Key Algorithm: rsaEncryption
    Signature Algorithm: md5WithRSAEncryption
```

**(4).验证请求文件的数字签名,这样可以验证出证书请求文件是否被篡改过。下面的命令中"-verify"选项表示验证证书请求文件的数字签名。**

```
[root@xuexi tmp]# openssl req -verify -in req2.csr
verify OK
-----BEGIN CERTIFICATE REQUEST-----
MIIBgDCB6gIBADBBMQswCQYDVQQGEwJDTjELMAkGA1UECAwCRkoxCzAJBgNVBAcM
AlhNMRgwFgYDVQQDDA93d3cueW91d2FudC5jb20wgZ8wDQYJKoZIhvcNAQEBBQAD
gY0AMIGJAoGBAMbx9bfsC0GTn7DijfGFs56Fb8atX9ABRDE/wmE74jXjdfbH4ZOg
Te0Orlu5pA4jqXDgSLzlQvjD6QsyhToyvtyQbgGSfXSVOPcgfAohDNo9t6+mnvs/
5rFQJ1+uI6gsLMbwQBJidLGnM1pOvFo2671Vm2jewDLVweGP5wmIfDyLAgMBAAGg
ADANBgkqhkiG9w0BAQQFAAOBgQCcvWuwmeAowbqLEsSpBVGnRfDEeH897v1r/SaX
9yYhpc3Kp5HKQ3LpSZBYGxlIsE6I3DMT5d1wcPeKRi8B6BIfemYOEbhLVGLmhNAg
iHyV/s1/TaOc31QZMY1HvD5BTOlhed+MpevWAFX2CRXuhKYBOimCrGNJxrFj4srJ
M1zDOA==
-----END CERTIFICATE REQUEST-----
```

结果中第一行的"verify OK"表示证书请求文件是完整未被篡改过的，但同时输出了证书请求的内容。如果不想输出这部分内容，使用"-noout"选项即可。

```
[root@xuexi tmp]# openssl req -verify -in req2.csr -noout
verify OK
```

**(5).自签署证书，可用于自建根CA时。**

使用openssl req自签署证书时，需要使用"-x509"选项，由于是签署证书请求文件，所以可以指定"-days"指定所颁发的证书有效期。

```
[root@xuexi tmp]# openssl req -x509 -key pri_key.pem -in req1.csr -out CA1.crt -days 365
```

由于openssl req命令的主要功能是创建和管理证书请求文件，所以没有提供对证书文件的管理能力，暂时也就只能通过cat来查看证书文件CA1.crt了。

```
[root@xuexi tmp]# cat CA1.crt
-----BEGIN CERTIFICATE-----
MIICUDCCAbmgAwIBAgIJAIrxQ+zicLzIMA0GCSqGSIb3DQEBBQUAMEExCzAJBgNV
BAYTAkNOMQswCQYDVQQIDAJGSjELMAkGA1UEBwwCWE0xGDAWBgNVBAMMD3d3dy55
b3V3YW50LmNvbTAeFw0xNzA2MjcwNzU0NTJaFw0xODA2MjcwNzU0NTJaMEExCzAJ
BgNVBAYTAkNOMQswCQYDVQQIDAJGSjELMAkGA1UEBwwCWE0xGDAWBgNVBAMMD3d3
dy55b3V3YW50LmNvbTCBnzANBgkqhkiG9w0BAQEFAAOBjQAwgYkCgYEAxvH1t+wL
QZOfsOKN8YWznoVvxq1f0AFEMT/CYTviNeN19sfhk6BN7Q6uW7mkDiOpcOBIvOVC
+MPpCzKFOjK+3JBuAZJ9dJU49yB8CiEM2j23r6ae+z/msVAnX64jqCwsxvBAEmJ0
saczWk68WjbrvVWbaN7AMtXB4Y/nCYh8PIsCAwEAAaNQME4wHQYDVR0OBBYEFMLa
Dm9yZeRh3Bu+zmpU2iKbQBQgMB8GA1UdIwQYMBaAFMLaDm9yZeRh3Bu+zmpU2iKb
QBQgMAwGA1UdEwQFMAMBAf8wDQYJKoZIhvcNAQEFBQADgYEAd2CJPe987RO34ySA
7EC0zQkhDz9d2vvvPWYjq0XA/frntlKKhgFwypWPBwwFTBwfvLHMnNpKy0zXXAkB
1ttgzMgka/qv/gcKoLN3dwM7Hz+eCl/cXVJmVG7PqAjfqSr6IyM7v/B6dC0Xv49m
h5mv24HqKtJoEeI0iARaNmOxKeE=
-----END CERTIFICATE-----
```

实际上，"-x509"选项和"-new"或"-newkey"配合使用时，可以不指定证书请求文件，它在自签署过程中将在内存中自动创建证书请求文件，当然，既然要创建证书请求文件，就需要人为输入申请者的信息了。例如：

```
[root@xuexi tmp]# openssl req -new -x509 -key pri_key.pem -out CA1.crt -days 365
```

其实，使用"-x509"选项后，"-new"或"-newkey"将表示创建一个证书文件而不是一个证书请求文件。

**(6).让openssl req自动创建所需的私钥文件。**

在前面的所有例子中，在需要私钥的时候都明确使用了"-key"选项提供私钥。其实如果不提供，**openssl req会在任何需要私钥的地方自动创建私钥**，并保存在特定的位置，默认的保存位置为当前目录，文件名为privkey.pem，具体保存的位置和文件名由配置文件(默认为/etc/pki/tls/openssl.cnf)决定，此处不讨论该文件。当然，openssl req命令的"-keyout"选项可以指定私钥保存位置。

例如：

```
[root@xuexi tmp]# openssl req -new -out req3.csr
Generating a 2048 bit RSA private key   # 自动创建私钥
..................+++
.....................................+++
writing new private key to 'privkey.pem'
Enter PEM pass phrase:     # <==要求输入加密私钥文件的密码，且要求长度为4-1024个字符
Verifying - Enter PEM pass phrase:
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:^C
```

但是，openssl req在自动创建私钥时，将总是加密该私钥文件，并提示输入加密的密码。可以使用"-nodes"选项禁止加密私钥文件。

```
[root@xuexi tmp]# openssl req -new -out req3.csr -nodes
Generating a 2048 bit RSA private key
.............+++
..........................+++
writing new private key to 'privkey.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:^C
```

指定自动创建私钥时，私钥文件的保存位置和文件名。使用"-keyout"选项。

```
[root@xuexi tmp]# openssl req -new -out req3.csr -nodes -keyout myprivkey.pem
Generating a 2048 bit RSA private key
......................+++
...........................+++
writing new private key to 'myprivkey.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:^C
```

**(7).使用"-newkey"选项。**

"-newkey"选项和"-new"选项类似，只不过"-newkey"选项可以直接指定私钥的算法和长度，所以它主要用在openssl req自动创建私钥时。

它的使用格式为`-newkey arg`，其中arg的格式为`rsa:numbits`，rsa表示创建rsa私钥，numbits表示私钥的长度，如果不给定长度(即`-newkey rsa`)则默认从配置文件中读取长度值。其实不止支持rsa私钥，只不过现在基本都是用rsa私钥，所以默认就使用rsa。

```
[root@xuexi tmp]# openssl req -newkey rsa:2048 -out req3.csr -nodes -keyout myprivkey.pem
Generating a 2048 bit RSA private key
....+++
.....................+++
writing new private key to 'myprivkey.pem'
-----
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:^C
```

<a name="blogopensslreq"></a>

通过上面一系类的举例说明后，想必openssl req的各基本选项的用法都通了。从上面的示例中也发现了，openssl req经常会依赖于配置文件(默认为/etc/pki/tls/openssl.cnf)中的值。所以，先将openssl req的命令用法总结下，再简单说明下配置文件中和req有关的内容。

```
openssl req [-new] [-newkey rsa:bits] [-verify] [-x509]
            [-in filename] [-out filename] [-key filename]
            [-passin arg] [-passout arg] [-keyout filename]
            [-pubkey] [-nodes] [-[dgst]] [-config filename]
            [-subj arg] [-days n] [-set_serial n]
            [-extensions section] [-reqexts section] [-utf8]
            [-nameopt] [-reqopt] [-subject] [-subj arg] [-text]
            [-noout] [-batch] [-verbose]
 
选项说明：
-new：创建一个证书请求文件，会交互式提醒输入一些信息，这些交互选项以及交互
      选项信息的长度值以及其他一些扩展属性在配置文件(默认为openssl.cnf，
      还有些辅助配置文件)中指定了默认值。如果没有指定"-key"选项，则会自动
      生成一个RSA私钥，该私钥的生成位置也在openssl.cnf中指定了。如果指定
      了-x509选项，则表示创建的是自签署证书文件，而非证书请求文件

-newkey args：类似于"-new"选项，创建一个新的证书请求，并创建私钥。args的格
    式是"rsa:bits"(其他加密算法请查看man)，其中bits是rsa密钥的长度，如果
    bits省略了(即-newkey rsa)，则长度根据配置文件中default_bits指令的值
    作为默认长度，默认该值为2048。如果指定了-x509选项，则表示创建的是自签
    署证书文件，而非证书请求文件

-nodes：默认情况下，openssl req自动创建私钥时都要求加密并提示输入加密密码，
        指定该选项后则禁止对私钥文件加密
        
-key filename：指定私钥的输入文件，创建证书请求时需要

-keyout filename：指定自动创建私钥时私钥的存放位置，若未指定该选项，则使用
                  配置文件中default_keyfile指定的值，默认该值为privkey.pem

-[dgst]：指定对创建请求时提供的申请者信息进行数字签名时的单向加密算法，
         如-md5/-sha1/-sha512等，若未指定则默认使用配置文件中default_md指定的值

-verify：对证书请求文件进行数字签名验证

-x509：指定该选项时，将生成一个自签署证书，而不是创建证书请求。一般用于测试
       或者为根CA创建自签名证书

-days n：指定自签名证书的有效期限，默认30天，需要和"-x509"一起使用。注意
         是自签名证书期限，而非请求的证书期限，因为证书的有效期是颁发者指
         定的，证书请求者指定有效期是没有意义的，配置文件中的default_days
         指定了请求证书的有效期限，默认365天

-set_serial n：指定生成自签名证书时的证书序列号，该序列号将写入配置文件中
        serial指定的文件中，这样就不需要手动更新该序列号文件支持数值和16进
        制值(0x开头)，虽然也支持负数，但不建议

-in filename：指定证书请求文件filename。注意，创建证书请求文件时是不需要
              指定该选项的

-out filename：证书请求或自签署证书的输出文件，也可以是其他内容的输出文件，
               不指定时默认stdout

-subj args：替换或自定义证书请求时需要输入的信息，并输出修改后的请求信息。
           args的格式为"/type0=value0/type1=value1..."，如果value为空，
           则表示使用配置文件中指定的默认值，如果value值为"."，则表示该项
           留空。其中可识别type(man req)有：C是Country、ST是state、
           L是localcity、O是Organization、OU是Organization Unit、
           CN是common name等
 
【输出内容选项：】
-text         ：以文本格式打印证书请求
-noout        ：不输出部分信息
-subject      ：输出证书请求文件中的subject(如果指定了x509，则打印证书中的subject)
-pubkey       ：输出证书请求文件中的公钥

【配置文件项和杂项：】
-passin arg：传递解密密码
-passout arg：指定加密输出文件时的密码
-config filename：指定req的配置文件，指定后将忽略所有的其他配置文件。
      如果不指定则默认使用/etc/pki/tls/openssl.cnf中req段落的值
-batch：非交互模式，直接从配置文件(默认/etc/pki/tls/openssl.cnf)
      中读取证书请求所需字段信息。但若不指定"-key"时，仍会询问key
-verbose：显示操作执行的详细信息
```

以下则是配置文件中(默认/etc/pki/tls/openssl.cnf)关于req段落的配置格式。

```
input_password：密码输入文件，和命令行的"-passin args"选项对应
output_password：密码的输出文件，与命令行的"-passout args"选项对应
default_bits：openssl req自动生成RSA私钥时的长度，不写时默认是512，
              命令行的"-new"和"-newkey"可能会用到它 
default_keyfile：默认的私钥输出文件，与命令行的"-keyout"选项对应 
encrypt_key：当设置为no时，自动创建私钥时不会加密该私钥。设置为no时
             与命令行的"-nodes"等价。还有等价兼容写法：encry_rsa_key 
default_md：指定创建证书请求时对申请者信息进行数字签名的单向加密算法，
            与命令行的"-[dgst]"对应 
prompt：当指定为no时，则不提示输入证书请求的字段信息，而是直接从
        openssl.cnf中读取。
        请小心设置该选项，很可能请求文件创建失败就是因为该选项设置为
        no distinguished_name：(DN)是一个扩展属性段落，用于指定证
        书请求时可被识别的字段名称。
```

其中`-passin args`以及`-passout args`传递密码的格式参考[openssl传递密码参数的格式][1]。

以下是默认的配置文件格式及值。关于配置文件的详细分析见下文[openssl配置文件](#openssl_cnf)部分。

```
[ req ]
default_bits            = 2048
default_md              = sha1
default_keyfile         = privkey.pem
distinguished_name      = req_distinguished_name
attributes              = req_attributes
x509_extensions = v3_ca # The extentions to add to the self signed cert
string_mask = utf8only
[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = XX
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
localityName                    = Locality Name (eg, city)
localityName_default    = Default City
0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = Default Company Ltd
organizationalUnitName          = Organizational Unit Name (eg, section)
commonName                      = Common Name (eg, your name or your server\'s hostname)
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64
```

<a name="openssl_cnf"></a>

## OpenSSL主配置文件openssl.cnf

虽说配置文件很多设置不用修改就能直接使用，但是了解它是配置openssl相关事项所必须的。而且要实现复杂多功能，必然要对配置相关了然于心。

### man config

该帮助文档说明了openssl.cnf以及一些其他辅助配置文件的规范、格式及读取方式。后文中的所有解释除非特别指明，都将以openssl.cnf为例。

```
[root@xuexi ~]# whatis config
Config (3pm) - access Perl configuration information
config (5ssl) - OpenSSL CONF library configuration files
Config::Extensions (3pm) - hash lookup of which core extensions were built
config.guess [config] (1) - guess the build system triplet
config [openssl] (5ssl) - OpenSSL CONF library configuration files
config.sub [config] (1) - validate and canonicalize a configuration triplet
config-util (5) - Common PAM configuration file for configuration utilities
```

因此直接`man config`即可。

配置文件openssl.cnf中分成了多个段落，每个段落都使用中括号包围的方式`[section_name]`来标识。section_name可以包含字母、数字和下划线。

第一个section被解释为默认段落，默认段落一般（是一般不是一定）没有`[section_name]`标识。当搜索某一个section时，将首先搜索有名称的section，然后还会搜索默认section，如果没有找到匹配的有名称的section，将直接读取默认section。

该配置文件中使用`#`开头来书写注释信息。每个section包含一些name以及它们的值，格式为`name=value`，name和value的前导或尾随空格被忽略，如果要包含空格应该使用引号包围。

在name部分可以包含字母、数字以及一些标点符号，如`. , ; _`。

在value部分可以使用变量扩展。在每个section中可以定义变量，每个section的变量默认只作用于当前section，变量引用的格式有两种`$var ${var}`。如果想要引用其他section中的变量或name，可以使用`$section_name::name`或`${section::name}`。

在value部分可以指定为其他section的指针。请参看下文的示例。

可以使用反斜线`\`转义，包括转义引号字符以及反斜线本身，也可以使用`\`来进入多行书写模式。另外`\n、\r、\b、\t`是能够被识别的。

以下为书写示例，注意其中的特性。

```
/* This is the default section.*/
HOME=/temp
RANDFILE= ${ENV::HOME}/.rnd
configdir=$ENV::HOME/config

[ section_one ]
default_value = section_three
/* Also you can refer section_name by character "@" */
default_value = @section_three

[ section_two ]
/* We are now in section two. *//* Quotes permit leading and trailing whitespace */
any = " any variable name "
other = A string that can \
cover several lines \
by including \\ characters
message = Hello World\n

[section_three]
greeting = $section_one::message
```

### /etc/pki/tls/openssl.cnf

该文件主要设置了证书请求、签名、crl相关的配置。主要相关的伪命令为ca和req。对于x509不用该配置文件。

该文件从功能结构上分为4个段落：默认段、ca相关的段、req相关的段、tsa相关的段。每个段中都以`name=value`的格式定义。

该文件中没有被引用的段被视为忽略段，不会起到任何作用。

每个段中可以书写哪些name以及它们的意义，可以man相关命令，如man ca可以查看ca相关段可以书写的name，man req可以查看req相关段可以书写的name。

#### 默认段

第一段是默认段，一般没有section\_name，但不是一定没有，可以自定义有名称的。

默认段中定义的是一些公共属性，当搜索一个给定名称的段时，将首先搜索有名称的段，当搜索不到匹配的段后会搜索默认段。

以下是默认段的内容。

```
HOME = .
RANDFILE = $ENV::HOME/.rnd
oid_section = new_oids
```

仅定义了当前目录变量，以及随机数的文件路径变量。

至于最后一行的`oid_section=new_oids`表示指向`[new_oids]`段。以下为`new_oids`段。oid是是对象标识符，干啥的我也不知道，反正没改过它。

```
[ new_oids ]
tsa_policy1 = 1.2.3.4.1
tsa_policy2 = 1.2.3.4.5.6
tsa_policy3 = 1.2.3.4.5.7
```

#### ca相关的段

这些段定义ca相关的控制选项。以下为ca相关段内容。其中黄底加粗黑字的为必须项，黄底加粗红字的为建议设置或建议修改的项。

```
####################################################################
[ ca ]
default_ca  = CA_default        /*The default ca section*/
####################################################################
[ CA_default ]

/* Where everything is kept */
/*  #### 这是第一个openssl目录结构中的目录 */
dir= /etc/pki/CA

/* Where the issued certs are kept(已颁发的证书路径，即CA或自签的) */
/* #### 这是第二个openssl目录结构中的目录，但非必须 */
certs= $dir/certs

/* Where the issued crl are kept(已颁发的crl存放目录) */
/*  #### 这是第三个openssl目录结构中的目录*/
crl_dir = $dir/crl

/* database index file */
database = $dir/index.txt 

/* 设置为yes则database文件中的subject列不能出现重复值 */
/* 即不能为subject相同的证书或证书请求签名*/
/* 建议设置为no，但为了保持老版本的兼容性默认是yes */
#unique_subject = no

/* default place for new certs(将来颁发的证书存放路径) */
/* #### 这是第四个openssl目录结构中的目录 */
new_certs_dir = $dir/newcerts 

/* The A certificate(CA自己的证书文件) */
certificate = $dir/cacert.pem

/* The current serial number(提供序列号的文件)*/
serial = $dir/serial

/* the current crl number(当前crl序列号) */
crlnumber = $dir/crlnumber

/* The current CRL(当前CRL) */
crl = $dir/crl.pem

/* The private key(签名时需要的私钥，即CA自己的私钥) */
private_key = $dir/private/cakey.pem  

/* private random number file(提供随机数种子的文件) */
RANDFILE    = $dir/private/.rand

/* The extentions to add to the cert(添加到证书中的扩展项) */
/* 以下两行是关于证书展示格式的，虽非必须项，但推荐设置。一般就如下格式不用修改 */
x509_extensions = usr_cert

/* Subject Name options*/
name_opt = ca_default

/* Certificate field options */
cert_opt = ca_default

/* 以下是copy_extensions扩展项，需谨慎使用 */

/* 生成证书时扩展项的copy行为，可设置为none/copy/copyall */
/* 不设置该name时默认为none */
/* 建议简单使用时设置为none或不设置，且强烈建议不要设置为copyall */
# copy_extensions = copy
# crl_extensions = crl_ext

/* how long to certify for(默认的证书有效期) */
default_days = 365   

/* how long before next CRL(CRL的有效期) */
default_crl_days= 30    

/* use public key default MD(默认摘要算法) */
default_md = default   

/* keep passed DN ordering(Distinguished Name顺序，一般设置为no */
/* 设置为yes仅为了和老版本的IE兼容)*/
preserve = no

/* 证书匹配策略,此处表示引用[ policy_match ]的策略 */
policy = policy_match 

/* 证书匹配策略定义了证书请求的DN字段(field)被CA签署时和CA证书的匹配规则 */
/* 对于CA证书请求，这些匹配规则必须要和父CA完全相同 */
[ policy_match ]
/* match表示请求中填写的该字段信息要和CA证书中的匹配 */
countryName = match
stateOrProvinceName = match
organizationName = match

/* optional表示该字段信息可提供可不提供 */
organizationalUnitName  = optional 

/* supplied表示该字段信息必须提供 */
commonName = supplied
emailAddress = optional

/* For the 'anything' policy*/
/* At this point in time, you must list all acceptable 'object' types. */

/* 以下是没被引用的策略扩展，只要是没被引用的都是被忽略的 */
[ policy_anything ]
countryName     = optional
stateOrProvinceName = optional
localityName        = optional
organizationName    = optional
organizationalUnitName  = optional
commonName      = supplied
emailAddress        = optional 

/* 以下是添加的扩展项usr_cert的内容*/
[ usr_cert ]
/* 基本约束，CA:FALSE表示该证书不能作为CA证书，即不能给其他人颁发证书*/
basicConstraints=CA:FALSE   
/* keyUsage = critical,keyCertSign,cRLSign  # 指定证书的目的，也就是限制证书的用法 */
/* 除了上面两个扩展项可能会修改下，其余的扩展项别管了，如下面的 */
nsComment  = "OpenSSL Generated Certificate" 
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid,issuer
```

#### req相关的段

```
[ req ]
/* 生成证书请求时用到的私钥的密钥长度 */
default_bits = 2048

/* 证书请求签名时的单向加密算法 */
default_md = sha1

/* 默认新创建的私钥存放位置， */
/* 如-new选项没指定-key时会自动创建私钥 */
/* -newkey选项也会自动创建私钥 */
default_keyfile = privkey.pem

/* 可识别的字段名(常被简称为DN) */
/* 引用req_distinguished_name段的设置 */
distinguished_name  = req_distinguished_name 

/* 加入到自签证书中的扩展项 */
x509_extensions = v3_ca
/* 加入到证书请求中的扩展项 */
# req_extensions = v3_req
/* 证书请求的属性，引用req_attributes段的设置，可以不设置它 */
attributes  = req_attributes

/* 自动生成的私钥文件要加密否？一般设置no，和-nodes选项等价 */
# encrypt_key = yes | no 
/* 输入和输出私钥文件的密码，如果该私钥文件有密码，不写该设置则会提示输入 */
/* input_password = secret */
/* output_password = secret */

/* 设置为no将不提示输入DN field，而是直接从配置文件中读取，*/
/* 需要同时设置DN默认值，否则创建证书请求时将出错 */
# prompt = yes | no 
string_mask = utf8only

[ req_distinguished_name ]
/* 以下项均可指定可不指定，但ca段的policy中指定为match和supplied一定要指定。 */
/* 以下选项都可以自定义，如countryName = C，commonName = CN */

countryName             = Country Name (2 letter code) /* 国家名(C) */
countryName_default     = XX /* 默认的国家名 */
countryName_min         = 2  /* 填写的国家名的最小字符长度 */
countryName_max         = 2  /* 填写的国家名的最大字符长度 */
stateOrProvinceName = State or Province Name (full name) /* 省份(S) */
/* stateOrProvinceName_default = Default Province */
localityName = Locality Name (eg, city) /* 城市(LT) */
localityName_default = Default City
0.organizationName  = Organization Name (eg, company) /* 公司(ON) */
0.organizationName_default  = Default Company Ltd
organizationalUnitName      = Organizational Unit Name (eg, section) /* 部门(OU) */
/* organizationalUnitName_default = */

/* 以下的commonName(CN)一般必须给,如果作为CA，那么需要在ca的policy中定义CN = supplied */
/* CN定义的是将要申请SSL证书的域名或子域名或主机名。 */
/* 例如要为zhonghua.com申请ssl证书则填写zhonghua.com，而不能填写www.zhonghua.com */
/* 要为www.zhonghua.com申请SSL则填写www.zhonghua.com */
/* CN必须和将要访问的网站地址一样，否则访问时就会给出警告 */
/* 该项要填写正确，否则该请求被签名后证书中的CN与实际环境中的CN不对应，将无法提供证书服务 */

/* 主机名(CN) */
commonName  = Common Name (eg, your name or your server\'s hostname) 
commonName_max  = 64
/* Email地址，很多时候不需要该项的 */
emailAddress = Email Address 
emailAddress_max = 64

/* 该段是为了某些特定软件的运行需要而设定的， */
/* 现在一般都不需要提供challengepassword */
/* 所以该段几乎用不上 */
/* 所以不用管这段 */
[ req_attributes ] 
challengePassword       = A challenge password
challengePassword_min   = 4
challengePassword_max   = 20
unstructuredName        = An optional company name
[ v3_req ]
/* Extensions to add to a certificate request */
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
[ v3_ca ]
/* Extensions for a typical CA */
subjectKeyIdentifier=hash
authorityKeyIdentifier=keyid:always,issuer
basicConstraints = CA:true

/* 典型的CA证书的使用方法设置，由于测试使用所以注释了 */
/* 如果真的需要申请为CA，那么该设置可以如此配置 */
# keyUsage = cRLSign, keyCertSign  
```

可以自定义DN(Distinguished Name)段中的字段信息，注意ca段中的policy指定的匹配规则中如果指定了match或这supplied的则DN中必须定义。例如下面的示例：由于只有countryName、organizationName和commonName被设定为match和supplied，其余的都是optional，所以在DN中可以只定义这3个字段，而且在DN中定义了自定义的名称。

```
[policy_to_match]
countryName = match
stateOrProvinceName = optional
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional
[DN]
countryName = "C"
organizationName = "O"
commonName = "Root CA"
```

#### 配置文件示例

以下是一个配置文件的示例。假设该配置文件路径为/ssl/ssl.conf。

```
[default]
name = root-ca    /* 变量*/
default_ca = CA_default
name_opt = ca_default
cert_opt = ca_default

[CA_default]
home = .     /* 变量*/
database = $home/db/index
serial = $home/db/serial
crlnumber = $home/db/crlnumber
certificate = $home/$name.crt
private_key = $home/private/$name.key
RANDFILE = $home/private/random
new_certs_dir = $home/certs
unique_subject = no
copy_extensions = none
default_days = 3650
default_crl_days = 365
default_md = sha256
policy = policy_to_match

[policy_to_match]
countryName = match
stateOrProvinceName = optional
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[CA_DN]
countryName = "C"
contryName_default = "CN"
organizationName = "O"
organizationName_default = "jmu"
commonName = "CN"
commonName_default = "longshuai.com"

[req]
default_bits = 4096
encrypt_key = no
default_md = sha256
utf8 = yes
string_mask = utf8only
# prompt = no  /* 测试时该选项导致出错，所以将其注释掉*/
distinguished_name = CA_DN
req_extensions = ca_ext

[ca_ext]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
```

根据该配置文件示例，进行自建根CA、签名等的操作方法请看：[openssl签署和自签署证书](#ca_sign)。

## openssl ca(签署和自建CA)

用于签署证书请求、生成吊销列表CRL以及维护已颁发证书列表和这些证书状态的数据库。因为一般人无需管理crl，所以本文只介绍openssl ca关于证书管理方面的功能。

证书请求文件使用CA的私钥签署之后就是证书，签署之后将证书发给申请者就是颁发证书。在签署时，为了保证证书的完整性和一致性，还应该对签署的证书生成数字摘要，即使用单向加密算法。

由于openssl ca命令对配置文件(默认为/etc/pki/tls/openssl.cnf)的依赖性非常强，所以建议结合前文[配置文件openssl.cnf](#openssl_cnf)来阅读，如果不明白配置文件，下面的内容很可能不知所云。

在配置文件中指定了签署证书时所需文件的结构，默认openssl.cnf中的结构要求如下：

```
[ CA_default ]
dir             = /etc/pki/CA             # 定义路径变量
certs           = $dir/certs              # 已颁发证书的保存目录
database        = $dir/index.txt          # 数据库索引文件
new_certs_dir   = $dir/newcerts           # 新签署的证书保存目录
certificate     = $dir/cacert.pem         # CA证书路径名
serial          = $dir/serial             # 当前证书序列号
private_key     = $dir/private/cakey.pem  # CA的私钥路径名
```

其中目录`/etc/pki/CA/{certs,newcerts,private}`在安装openssl后就默认存在，所以无需独立创建，但证书的database文件index.txt和序列文件serial必须创建好，且序列号文件中得先给定一个序号，如"01"。

```
[root@xuexi tmp]# touch /etc/pki/CA/index.txt 

[root@xuexi tmp]# echo "01" > /etc/pki/CA/serial
```

另外，要签署证书请求，需要CA自己的私钥文件以及CA自己的证书，先创建好CA的私钥，存放位置为配置文件中private\_key所指定的值，默认为/etc/pki/CA/private/cakey.pem。

```
[root@xuexi tmp]# openssl genrsa -out /etc/pki/CA/private/cakey.pem
```

**(1).使用openssl ca自建CA**

要提供CA自己的证书，测试环境下CA只能自签署，使用`openssl req -x509`、`openssl x509`和`openssl ca`都可以自签署证书请求文件，此处仅介绍`openssl ca`命令自身自签署的方法。

先创建CA的证书请求文件，建议使用CA的私钥文件/etc/pki/CA/private/cakey.pem来创建待自签署的证书请求文件，虽非必须，但方便管理。创建请求文件时，其中Country Name、State or Province Name、Organization Name和Common Name默认是必须提供的。

```
[root@xuexi tmp]# openssl req -new -key /etc/pki/CA/private/cakey.pem -out rootCA.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:FJ
Locality Name (eg, city) [Default City]:XM
Organization Name (eg, company) [Default Company Ltd]:JM
Organizational Unit Name (eg, section) []:IT
Common Name (eg, your name or your server's hostname) []:www.iwant.com
Email Address []:.
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:.
An optional company name []:.
```

然后使用openssl ca命令自签署该证书请求文件。如果有两次交互式询问则表示自签署将成功，如果失败，则考虑数据库文件index.txt是否创建、序列号文件serial是否存在且有序号值、私钥文件cakey.pem是否路径正确、创建证书请求文件时是否该提供的没有提供等情况。

```
[root@xuexi tmp]# openssl ca -selfsign -in rootCA.csr
# 默认采用/etc/pki/tls/openssl.cnf作为配置文件
Using configuration from /etc/pki/tls/openssl.cnf
# 验证证书请求文件的数字签名，确保该证书请求文件是完整未修改过的
Check that the request matches the signature
Signature ok
Certificate Details:   待生成证书的信息
        Serial Number: 1 (0x1)    # 序列号为1
        Validity
            # 证书有效期起始日为2017-6-17 10:06:29
            Not Before: Jun 27 10:06:29 2017 GMT
            # 证书有效期终止日为2018-6-17 10:06:29
            Not After : Jun 27 10:06:29 2018 GMT
        Subject:       # Subject信息，subject是非常重要的信息
            countryName               = CN
            stateOrProvinceName       = FJ
            organizationName          = JM
            organizationalUnitName    = IT
            commonName                = www.iwant.com
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Comment:
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier:
                A5:0D:DD:D6:47:C6:24:74:20:F4:62:77:F6:A9:63:3E:52:D2:8A:66
            X509v3 Authority Key Identifier:
                keyid:A5:0D:DD:D6:47:C6:24:74:20:F4:62:77:F6:A9:63:3E:52:D2:8A:66
 
Certificate is to be certified until Jun 27 10:06:29 2018 GMT (365 days)
Sign the certificate? [y/n]:y
 
 
1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries  # 向数据库文件添加一条该证书的记录
Certificate:     # 该证书的信息
    Data:
        Version: 3 (0x2)
        Serial Number: 1 (0x1)
    Signature Algorithm: sha1WithRSAEncryption
        Issuer: C=CN, ST=FJ, O=JM, OU=IT, CN=www.iwant.com
        Validity
            Not Before: Jun 27 10:06:29 2017 GMT
            Not After : Jun 27 10:06:29 2018 GMT
        Subject: C=CN, ST=FJ, O=JM, OU=IT, CN=www.iwant.com
        Subject Public Key Info:
            Public Key Algorithm: rsaEncryption
                Public-Key: (1024 bit)
                Modulus:
                    00:94:49:33:f4:90:a4:fc:a4:6b:65:75:4c:be:4f:
                    d1:3f:95:bd:24:60:c8:45:f9:eb:00:31:ac:45:6b:
                    ae:bb:63:bf:f2:a3:0c:e3:d3:50:20:33:1e:d9:e1:
                    8a:49:42:c6:e0:67:6d:3a:cb:2f:9c:90:ab:4c:10:
                    7a:4a:82:e1:6e:a0:6a:63:84:56:1c:a2:5f:11:60:
                    99:e0:cd:20:68:e9:98:40:68:c2:43:7c:97:12:ee:
                    31:8e:b1:73:7d:36:99:97:49:31:50:c1:8c:47:10:
                    16:f9:5d:37:11:00:73:3b:01:62:9b:36:36:97:08:
                    48:31:93:56:3f:6a:d9:a6:99
                Exponent: 65537 (0x10001)
        X509v3 extensions:
            X509v3 Basic Constraints:
                CA:FALSE
            Netscape Comment:
                OpenSSL Generated Certificate
            X509v3 Subject Key Identifier:
                A5:0D:DD:D6:47:C6:24:74:20:F4:62:77:F6:A9:63:3E:52:D2:8A:66
            X509v3 Authority Key Identifier:
                keyid:A5:0D:DD:D6:47:C6:24:74:20:F4:62:77:F6:A9:63:3E:52:D2:8A:66
 
    Signature Algorithm: sha1WithRSAEncryption
         1e:4e:f4:e4:c9:33:52:85:69:ae:b4:2a:37:37:44:90:9b:52:
         b3:e9:89:1c:b2:f2:17:41:d8:05:02:63:9a:4f:64:4d:c9:ce:
         0c:81:48:22:4f:73:8a:4c:f7:b8:bf:64:b2:77:8a:2e:43:80:
         39:03:de:27:19:09:d2:88:39:11:8f:8b:4b:37:c0:12:68:ef:
         79:5b:28:d4:cf:c9:b8:e1:77:24:6e:b4:5b:83:4a:46:49:a1:
         ad:5c:b7:d8:da:49:9a:45:73:b9:8e:eb:1a:9c:2e:6c:70:d3:
         c5:db:9c:46:02:59:42:bf:ad:bc:21:4c:d1:6b:6b:a7:87:33:
         1a:6b
-----BEGIN CERTIFICATE-----
MIICiTCCAfKgAwIBAgIBATANBgkqhkiG9w0BAQUFADBMMQswCQYDVQQGEwJDTjEL
MAkGA1UECAwCRkoxCzAJBgNVBAoMAkpNMQswCQYDVQQLDAJJVDEWMBQGA1UEAwwN
d3d3Lml3YW50LmNvbTAeFw0xNzA2MjcxMDA2MjlaFw0xODA2MjcxMDA2MjlaMEwx
CzAJBgNVBAYTAkNOMQswCQYDVQQIDAJGSjELMAkGA1UECgwCSk0xCzAJBgNVBAsM
AklUMRYwFAYDVQQDDA13d3cuaXdhbnQuY29tMIGfMA0GCSqGSIb3DQEBAQUAA4GN
ADCBiQKBgQCUSTP0kKT8pGtldUy+T9E/lb0kYMhF+esAMaxFa667Y7/yowzj01Ag
Mx7Z4YpJQsbgZ206yy+ckKtMEHpKguFuoGpjhFYcol8RYJngzSBo6ZhAaMJDfJcS
7jGOsXN9NpmXSTFQwYxHEBb5XTcRAHM7AWKbNjaXCEgxk1Y/atmmmQIDAQABo3sw
eTAJBgNVHRMEAjAAMCwGCWCGSAGG+EIBDQQfFh1PcGVuU1NMIEdlbmVyYXRlZCBD
ZXJ0aWZpY2F0ZTAdBgNVHQ4EFgQUpQ3d1kfGJHQg9GJ39qljPlLSimYwHwYDVR0j
BBgwFoAUpQ3d1kfGJHQg9GJ39qljPlLSimYwDQYJKoZIhvcNAQEFBQADgYEAHk70
5MkzUoVprrQqNzdEkJtSs+mJHLLyF0HYBQJjmk9kTcnODIFIIk9zikz3uL9ksneK
LkOAOQPeJxkJ0og5EY+LSzfAEmjveVso1M/JuOF3JG60W4NKRkmhrVy32NpJmkVz
uY7rGpwubHDTxducRgJZQr+tvCFM0Wtrp4czGms=
-----END CERTIFICATE-----
Data Base Updated
```

自签署成功后，在/etc/pki/CA目录下将生成一系列文件。

```
[root@xuexi tmp]# tree -C /etc/pki/CA
/etc/pki/CA
├── certs
├── crl
├── index.txt
├── index.txt.attr
├── index.txt.old
├── newcerts
│   └── 01.pem
├── private
│   └── cakey.pem
├── serial
└── serial.old
```

其中newcerts目录下的01.pem即为刚才自签署的证书文件，因为它是CA自身的证书，所以根据配置文件中的`certificate=$dir/cacert.pem`项，应该将其放入/etc/pki/CA目录下，且命名为cacert.pem，只有这样以后才能签署其它证书请求。

```
[root@xuexi tmp]# cp /etc/pki/CA/newcerts/01.pem /etc/pki/CA/cacert.pem
```

至此，自建CA就完成了，查看下数据库索引文件和序列号文件。

```
[root@xuexi tmp]# cat /etc/pki/CA/index.txt
V   180627100629Z    01  unknown /C=CN/ST=FJ/O=JM/OU=IT/CN=www.iwant.com

[root@xuexi tmp]# cat /etc/pki/CA/serial
02
```

那么，下次签署证书请求时，序列号将是"02"。

将上述自建CA的过程总结如下：

```
[root@xuexi tmp]# touch /etc/pki/CA/index.txt 
[root@xuexi tmp]# echo "01" > /etc/pki/CA/serial
[root@xuexi tmp]# openssl genrsa -out /etc/pki/CA/private/cakey.pem
[root@xuexi tmp]# openssl req -new -key /etc/pki/CA/private/cakey.pem -out rootCA.csr
[root@xuexi tmp]# openssl ca -selfsign -in rootCA.csr
[root@xuexi tmp]# cp /etc/pki/CA/newcerts/01.pem /etc/pki/CA/cacert.pem
```

以上过程是完全读取默认配置文件创建的，其实很多过程是没有那么严格的，openssl ca命令自身可以指定很多选项覆盖配置文件中的项，但既然提供了默认的配置文件及目录结构，为了方便管理，仍然建议完全采用配置文件中的项。

**(2).为他人颁发证书。**

首先申请者创建一个证书请求文件。

```
[root@xuexi tmp]# openssl req -new -key privatekey.pem -out youwant1.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:CN
State or Province Name (full name) []:FJ
Locality Name (eg, city) [Default City]:XM
Organization Name (eg, company) [Default Company Ltd]:JM
Organizational Unit Name (eg, section) []:.
Common Name (eg, your name or your server's hostname) []:www.youwant.com
Email Address []:.
 
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:.
An optional company name []:.
```

其中Country Name、State or Province Name、Organization Name和Common Name必须提供，且前三者必须和CA的subject中的对应项完全相同。这些是由配置文件中的匹配策略决定的。

```
[ ca ]
default_ca      = CA_default            # The default ca section
[ CA_default ]
policy          = policy_match
[ policy_match ]
countryName             = match
stateOrProvinceName     = match
organizationName        = match
organizationalUnitName  = optional
commonName              = supplied
emailAddress            = optional
```

"match"表示openssl ca要签署的证书请求文件中的项要和CA证书中的项匹配，即要相同，"supplied"表示必须要提供的项，"optional"表示可选项，所以可以留空。

现在就可以将证书请求文件发送给CA，让CA帮忙签署。

```
[root@xuexi tmp]# openssl ca -in youwant1.csr
```

签署成功后，查看下/etc/pki/CA下的文件结构。

```
[root@xuexi tmp]# tree -C /etc/pki/CA/
/etc/pki/CA/
├── cacert.pem
├── certs
├── crl
├── index.txt
├── index.txt.attr
├── index.txt.attr.old
├── index.txt.old
├── newcerts
│   ├── 01.pem
│   └── 02.pem
├── private
│   └── cakey.pem
├── serial
└── serial.old
 
4 directories, 10 files
```

其中"02.pem"就是刚才签署成功的证书，将此证书发送给申请者即表示颁发完成。

再看下数据库索引文件和序列号文件。

```
[root@xuexi tmp]# cat /etc/pki/CA/index.txt
V   180627100629Z  01  unknown /C=CN/ST=FJ/O=JM/OU=IT/CN=www.iwant.com
V   180627110022Z  02  unknown /C=CN/ST=FJ/O=JM/CN=www.youwant.com

[root@xuexi tmp]# cat /etc/pki/CA/serial
03
```

**(3).openssl ca命令用法**

经过上面的示例，应该对`openssl ca`命令的用法大致了解了，下面是其完整的用法说明，不包括crl相关功能。

```
openssl ca [-verbose] [-config filename] [-name section] [-startdate date] [-enddate date] [-days arg] [-md arg] [-policy arg] [-keyfile arg] [-key arg] [-passin arg] [-cert file]
[-selfsign] [-in file] [-out file] [-notext] [-outdir dir] [-infiles] [-ss_cert file] [-preserveDN] [-noemailDN] [-batch] [-extensions section] [-extfile section] [-subj arg] [-utf8]
```

要注意，ca命令是用于签署证书的，所以它所需要的文件除了配置文件外就是私钥文件和证书请求文件，而签名后生成的文件是证书文件，因此使用"-in"指定的对象是待签署文件，"-infiles"则是指定多个待签署文件，"-keyfile"是指定私钥文件，"-out"是指定输出的证书文件。

```
【选项说明：】
-config filename ：指定要使用的配置文件，指定后将忽略openssl.cnf中指定的关于ca的配置选项。
-name section    ：指定使用配置文件中的那个section。指定后将忽略openssl.cnf中的default_ca段。
-in filename     ：指定要被CA签署的单个证书请求文件。根CA为其他证书签署时使用。
-infiles         ：该选项只能是最后一个选项，该选项所接的所有参数都被认为是要
                   被签署的证书请求文件，即一次性签署多个请求文件时使用的选项。
-selfsign        ：自签署。指定-ss_cert选项时该选项被忽略。
-ss_cert filename：将被CA自签署的单个证书文件。也就是说要重新签署证书。
-out filename    ：证书的输出文件，同时也会输出到屏幕。不指定时默认输出到stdout。
-outdir dir_name ：证书的输出目录。指定该选项时，将自动在此目录下生成一个文件名
                   包含16进制serial值的".pem"证书文件。
-cert            ：CA自己的证书文件。
-keyfile filename：指定签署证书请求时的私钥文件，即CA自己的私钥文件。
-key passwd_value：指定私钥的加密密码。
-passin arg      ：传递解密密码
-verbose         ：打印操作执行时的详细信息
-notext          ：禁止以文本格式将证书输出到"-out"指定的文件中
-days arg        ：证书有效期限，从创建时刻开始算startdate，有效期结束点为enddate。
-startdate       ：自定义证书的开始时间，和"-enddate"一起使用可以推算出证书有效期。
-enddate         ：自定义证书的结束时间。
-md alg          ：指定单向加密算法
-policy arg      ：该选项是配置文件中的section内容，该选项指定了证书信息中的field
                   部分是否需要强制提供还是要强制匹配，或者可提供可不提供。
                   详细的见配置文件说明。
-extensions section：指定当前创建的证书使用配置文件中的哪个section作为扩展属性。
-batch           ：签署时使用批处理模式，即非交互模式。该模式下不会有两次询问
                   (是否签署、是否提交)。
-subj arg        ：替换证书请求中的subject，格式/type0=value0/type1=value1/type2=...
```

配置文件关于ca的部分，其中被标记为必须项的表示配置文件中或者命令行中必须给出该选项及其值。

```
new_certs_dir    ：等同于"-outdir"选项。必须项
certificat       ：等同于"-cert"选项，CA自己的证书文件。必须项
private_key      ：等同于"-keyfile"选项，签署证书请求文件时的私钥文件，即CA自己的私钥文件。必须项
default_days     ：等同于"-days"选项
default_startdate：等同于"-startdate"选项。
default_enddate  ：等同于"-enddate"选项。
default_md       ：等同于"-md"选项。必须项
database         ：openssl维护的数据库文件。存放证书条目信息及状态信息。必须项
serial           ：已颁发证书的序列号(16进制)文件。必须项且该文件中必须存在一个序列值
unique_subject   ：如果设置为yes，database中的subject列值必须不重复。
                   如果设置为no，允许subject重复。默认是yes，这是为了兼容老版本的
                   openssl，推荐设置为no。
x509_extensions  ：等同于"-extensions"选项。
policy           ：等同于"-policy"选项。必须项
name_opt/cert_opt：证书的展示格式，虽非必须但建议设置为ca_default，若不设置将默认
                   使用老版本的证书格式(不建议如此)。伪命令ca无法直接设置这两个选
                   项，而伪命令x509的"-nameopt"和"-certopt"选项可以分别设置。
copy_extensions  ：决定证书请求中的扩展项如何处理的。如果设置为none或不写该选项，
                   则扩展项被忽略并且不复制到证书中去。如果设置为copy，则证书请
                   求中已存在而证书中不存在的扩展项将复制到证书中。如果设置为
                   copyall，则证书请求中所有的扩展项都复制到证书中，此时若证书中
                   已存在某扩展项，则先删除再复制。
                   该选项的主要作用是允许证书请求为特定的扩展项如subjectAltName提
                   供值。使用该选项前请先查看man ca中的WARNINGS部分。建议一般简单
                   使用时设置为none或不设置。
```

## openssl x509(签署和自签署)

**主要用于输出证书信息，也能够签署证书请求文件、自签署、转换证书格式等。**

`openssl x509`工具**不会使用openssl配置文件中的设定，而是完全需要自行设定或者使用该伪命令的默认值，它就像是一个完整的小型的CA工具箱**。

```
openssl x509 [-in filename] [-out filename] [-serial] [-hash]
             [-subject_hash] [-issuer_hash] [-subject] [-issuer] 
             [-nameopt option] [-email] [-startdate] [-enddate]
             [-purpose] [-dates] [-modulus] [-pubkey] [-fingerprint]
             [-noout] [-days arg] [-set_serial n] [-signkey filename]
             [-x509toreq] [-req] [-CA filename] [-CAkey filename]
             [-CAcreateserial] [-CAserial filename] [-text]
             [-md2|-md5|-sha1|-mdc2] [-extfile filename]
             [-extensions section]
```

选项非常多，所以分段解释。

```
【输入输出选项：】
-in filename  ：指定证书输入文件，若同时指定了"-req"选项，则表示输入文件为证书请求文件。
-out filename ：指定输出文件
-md2|-md5|-sha1|-mdc2：指定单向加密的算法。
【信息输出选项：】
-text：以text格式输出证书内容，即以最全格式输出，
       包括public key,signature algorithms,issuer和subject names,
       serial number以及any trust settings.
-certopt option：自定义要输出的项
-noout         ：禁止输出证书请求文件中的编码部分
-pubkey        ：输出证书中的公钥
-modulus       ：输出证书中公钥模块部分
-serial        ：输出证书的序列号
-subject       ：输出证书中的subject
-issuer        ：输出证书中的issuer，即颁发者的subject
-subject_hash  ：输出证书中subject的hash码
-issuer_hash   ：输出证书中issuer(即颁发者的subject)的hash码
-hash          ：等价于"-subject_hash"，但此项是为了向后兼容才提供的选项
-email         ：输出证书中的email地址，如果有email的话
-startdate     ：输出证书有效期的起始日期
-enddate       ：输出证书有效期的终止日期
-dates         ：输出证书有效期，等价于"startdate+enddate"
-fingerprint   ：输出指纹摘要信息
```

输出证书某些信息的时候，可以配合"-noout"选项，然后再指定某些项来使用。例如：

```
[root@xuexi ~]# openssl x509 -in cert.pem -noout -text
[root@xuexi ~]# openssl x509 -in cert.pem -noout -serial
[root@xuexi ~]# openssl x509 -in cert.pem -noout -subject
[root@xuexi ~]# openssl x509 -in cert.pem -noout -issuer
[root@xuexi ~]# openssl x509 -in cert.pem -noout -fingerprint
[root@xuexi ~]# openssl x509 -in cert.pem -noout -issuer_hash
[root@xuexi ~]# openssl x509 -in cert.pem -noout -startdate -enddate
【签署选项：】
*********************************************************
*  子命令x509可以像openssl ca一样对证书或请求执行签名动作。 *
*  注意，openssl x509不读取配置文件，所有的一切配置都由x509 *
*  自行提供不读取配置文件，所有的一切配置都由x509自行提供，  *
*  所以openssl x509像是一个"mini CA"                     *
*********************************************************
-signkey filename：该选项用于提供自签署时的私钥文件，自签署的输入
    文件"-in file"的file可以是证书请求文件，也可以是已签署过的证书。
-clrext：从证书文件中删除所有的扩展项。
-days arg：指定证书有效期限，默认30天。
-x509toreq：将已签署的证书转换回证书请求文件。需要使用"-signkey"选
            项来传递需要的私钥。
-req：x509工具默认以证书文件做为inputfile(-in file)，指定该选项将使
      得input file的file为证书请求文件。
-set_serial n：指定证书序列号。该选项可以和"-singkey"或"-CA"选项一起使用。
      如果和"-CA"一起使用，则"-CAserial"或"-CAcreateserial"选项指定的
      serial值将失效。序列号可以使用数值或16进制值(0x开头)。也接受负值，
      但是不建议。
-CA filename：指定签署时所使用的CA证书。该选项一般和"-req"选项一起使用，
              用于为证书请求文件签署。
-CAkey filename：设置CA签署时使用的私钥文件。如果该选项没有指定，将
                 假定CA私钥已经存在于CA自签名的证书文件中。
-CAserial filename：设置CA使用的序列号文件。当使用"-CA"选项来签名时，
      它将会使用某个文件中指定的序列号来唯一标识此次签名后的证书文件。
      这个序列号文件的内容仅只有一行，这一行的值为16进制的数字。当某个
      序列号被使用后，该文件中的序列号将自动增加。默认序列号文件以CA证
      书文件基名加".srl"为后缀命名。如CA证书为"mycert.pem"，则默认寻
      找的序列号文件为"mycert.srl"
-CAcreateserial：当使用该选项时，如果CA使用的序列号文件不存在将自动
      创建：该文件将包含序列号值"02"并且此次签名后证书文件序列号为1。
      一般如果使用了"-CA"选项而序列号文件不存在将会产生错误"找不到srl文件"。
-extfile filename ：指定签名时包含要添加到证书中的扩展项的文件。
【CERTIFICATE EXTENSIONS】
-purpose：选项检查证书的扩展项并决定该证书允许用于哪些方面，即证书使用目的范围。
basicConstraints：该扩展项用于决定证书是否可以当作CA证书。
     格式为basicConstraints=CA:true | false
     1.如果CA的flag设置为true，那么该证书允许作为一个CA证书，即可以颁发下级证书或进行签名；
     2.如果CA的flag设置为false，那么该证书就不能作为CA，不能为下级颁发证书或签名；
     3.所有CA的证书中都必须设置CA的flag为true。
     4.如果basicConstraints扩展项未设置，那么证书被认为可疑的CA，即"possible CA"。
keyUsage：该扩展项用于指定证书额外的使用限制，即也是使用目的的一种表现方式。
     1.如果keyUsage扩展项被指定，那么该证书将又有额外的使用限制。
     2.CA证书文件中必须至少设置keyUsage=keyCertSign。
     3.如果设置了keyUsage扩展项，那么不论是否使用了critical，都将被限制在指定的
       使用目的purpose上。
```

例如，使用x509工具自建CA。由于x509无法建立证书请求文件，所以只能使用openssl req来生成请求文件，然后使用x509来自签署。自签署时，使用"-req"选项明确表示输入文件为证书请求文件，否则将默认以为是证书文件，再使用"-signkey"提供自签署时使用的私钥。

```
[root@xuexi ssl]# openssl req -new -keyout key.pem -out req.csr

[root@xuexi ssl]# openssl x509 -req -in req.csr -signkey key.pem -out x509.crt
```

x509也可以用来签署他人的证书请求，即为他人颁发证书。注意，为他人颁发证书时，确保serial文件存在，建议使用自动创建的选项"-CAcreateserial"。

```
[root@xuexi ssl]# openssl x509 -req -in req.csr -CA ca.crt -CAkey ca.key -out x509.crt -CAcreateserial
```

<a name="ca_sign"></a>

## openssl证书签署和自签署：自定义配置文件的实现方式

如非必要，建议采用默认配置文件方式。参考后文。

### 1.自建CA

自建CA的机制：1.生成私钥；2.创建证书请求；3.使用私钥对证书请求签名。

由于测试环境，所以自建的CA只能是根CA。所使用的配置文件如下。

```
[default]
name = root-ca    /* 变量*/
default_ca = CA_default
name_opt = ca_default
cert_opt = ca_default

[CA_default]
home = .     /* 变量*/
database = $home/db/index
serial = $home/db/serial
crlnumber = $home/db/crlnumber
certificate = $home/$name.crt
private_key = $home/private/$name.key
RANDFILE = $home/private/random
new_certs_dir = $home/certs
unique_subject = no
copy_extensions = none
default_days = 3650
default_crl_days = 365
default_md = sha256
policy = policy_to_match

[policy_to_match]
countryName = match
stateOrProvinceName = optional
organizationName = match
organizationalUnitName = optional
commonName = supplied
emailAddress = optional

[CA_DN]
countryName = "C"
contryName_default = "CN"
organizationName = "O"
organizationName_default = "jmu"
commonName = "CN"
commonName_default = "longshuai.com"

[req]
default_bits = 4096
encrypt_key = no
default_md = sha256
utf8 = yes
string_mask = utf8only
# prompt = no  /* 测试时该选项导致出错，所以将其注释掉*/
distinguished_name = CA_DN
req_extensions = ca_ext

[ca_ext]
basicConstraints = critical,CA:true
keyUsage = critical,keyCertSign,cRLSign
subjectKeyIdentifier = hash
```

创建配置文件，内容如上：

```
[root@xuexi ~]# mkdir /ssl; touch /ssl/ssl.conf

[root@xuexi ~]# cd /ssl

[root@xuexi ssl]# vim ssl.conf
```

创建openssl的目录结构中的目录，在上述配置文件中的目录分别为/ssl/db、/ssl/private和/ssl/certs，可以考虑将private目录的权限设置为600或者400。

```
[root@xuexi ssl]# mkdir /ssl/{db,private,certs}

[root@xuexi ssl]# chmod -R 400 private/
```

### 2.CA自签名

普通的证书请求需要使用CA的私钥进行签名变成证书，既然是自签名证书那当然是使用自己的私钥来签名。可以使用伪命令req、ca、x509来自签名。

#### 使用req伪命令创建CA

这里有两种方法：

- 1.一步完成，即私钥、证书请求、自签名都在一个命令中完成
- 2.分步完成，先生成私钥、再创建证书请求、再指定私钥来签名。

方法2中其实生成私钥和证书申请可以合并在一步中完成，证书申请和签名也可以合并在一步中完成。

方法一：一步完成

在下面的一步命令中，使用-new由于没有指定私钥输出位置，所以自动保存在ssl.conf中default\_keyfile指定的private.pem中；由于ssl.conf中的req段设置了encrypt\_key=no，所以交互时不需要输入私钥的加密密码；由于使用`req -x509`自签名的证书有效期默认为30天，而配置文件中req段又不能配置该期限，所以只能使用-days来指定有效期限，注意这个-days选项只作用于x509签名，证书请求中如果指定了时间是无效的。

```
[root@xuexi ssl]# openssl req -x509 -new -out req.crt -config ssl.conf -days 365
[root@xuexi ssl]# ll 
total 24
drwxr-xr-x 2 root root 4096 Nov 22 09:05 certs
drwxr-xr-x 2 root root 4096 Nov 22 09:05 db
drwx------ 2 root root 4096 Nov 22 09:05 private
-rw-r--r-- 1 root root 3272 Nov 22 10:52 private.pem  /* 注意权限为644 */
-rw-r--r-- 1 root root 1753 Nov 22 10:52 req.crt
-rw-r--r-- 1 root root 1580 Nov 22 10:51 ssl.conf
[root@xuexi ssl]# openssl x509 -noout -dates -in req.crt 
notBefore=Nov 22 02:52:24 2016 GMT
notAfter=Nov 22 02:52:24 2017 GMT
```

方法二：分步完成，这里把各种可能的步骤合并都演示一遍

【创建私钥和证书请求合并而签名独自进行的方法】

```
[root@xuexi ssl]# openssl req -newkey rsa:1024 -keyout key.pem -out req1.csr -config ssl.conf -days 365
[root@xuexi ssl]# openssl req -x509 -in req1.csr -key key.pem -out req1.crt
[root@xuexi ssl]# openssl x509 -noout -dates -in req1.crt /* 注意签名不要配置文件 */
notBefore=Nov 22 02:58:25 2016 GMT
notAfter=Dec 22 02:58:25 2016 GMT  /* 可以看到证书请求中指定-days是无效的 */
[root@xuexi ssl]# ll
total 36
drwxr-xr-x 2 root root 4096 Nov 22 09:05 certs
drwxr-xr-x 2 root root 4096 Nov 22 09:05 db
-rw-r--r-- 1 root root  912 Nov 22 10:57 key.pem
drwx------ 2 root root 4096 Nov 22 09:05 private
-rw-r--r-- 1 root root 3272 Nov 22 10:52 private.pem
-rw-r--r-- 1 root root  826 Nov 22 10:58 req1.crt
-rw-r--r-- 1 root root  688 Nov 22 10:57 req1.csr
-rw-r--r-- 1 root root 1753 Nov 22 10:52 req.crt
-rw-r--r-- 1 root root 1580 Nov 22 10:51 ssl.conf
```

【独自生成私钥，而请求和签名合并的方法】

```
[root@xuexi ssl]# (umask 077;openssl genrsa -out key1.pem 1024)
[root@xuexi ssl]# openssl req -x509 -new -key key1.pem -out req2.crt -config ssl.conf -days 365
[root@xuexi ssl]# openssl x509 -noout -dates -in req2.crt 
notBefore=Nov 22 03:28:31 2016 GMT
notAfter=Nov 22 03:28:31 2017 GMT
[root@xuexi ssl]# ll
total 44
drwxr-xr-x 2 root root 4096 Nov 22 09:05 certs
drwxr-xr-x 2 root root 4096 Nov 22 09:05 db
-rw-r--r-- 1 root root  912 Nov 22 10:57 key1.pem
-rw------- 1 root root  887 Nov 22 11:26 key2.pem
drwx------ 2 root root 4096 Nov 22 09:05 private
-rw-r--r-- 1 root root 3272 Nov 22 10:52 private.pem
-rw-r--r-- 1 root root  826 Nov 22 10:58 req1.crt
-rw-r--r-- 1 root root  688 Nov 22 10:57 req1.csr
-rw-r--r-- 1 root root  709 Nov 22 11:28 req2.crt
-rw-r--r-- 1 root root 1753 Nov 22 10:52 req.crt
-rw-r--r-- 1 root root 1580 Nov 22 10:51 ssl.conf
```

【完全分步进行】

```
[root@xuexi ssl]# rm -rf key* req* private.pem
[root@xuexi ssl]# (umask 077;openssl genrsa -out key.pem 1024)
[root@xuexi ssl]# openssl req -new -key key.pem -out req.csr -config ssl.conf 
[root@xuexi ssl]# openssl req -x509 -key key.pem -in req.csr -out req.crt -days 365
[root@xuexi ssl]# openssl x509 -noout -dates -in req.crt
notBefore=Nov 22 04:29:21 2016 GMT
notAfter=Nov 22 04:29:21 2017 GMT
[root@xuexi ssl]# ll
total 28
drwxr-xr-x 2 root root 4096 Nov 22 09:05 certs
drwxr-xr-x 2 root root 4096 Nov 22 09:05 db
-rw------- 1 root root  887 Nov 22 12:28 key.pem
drwx------ 2 root root 4096 Nov 22 09:05 private
-rw-r--r-- 1 root root  826 Nov 22 12:29 req.crt
-rw-r--r-- 1 root root  688 Nov 22 12:28 req.csr
-rw-r--r-- 1 root root 1580 Nov 22 10:51 ssl.conf
```

在本节的开头说明了创建证书请求时需要提供私钥，这个私钥的作用是为了提供公钥。下面是验证。

```
/* 提取私钥key.pem中的公钥到key.pub文件中 */
[root@xuexi ssl]# openssl rsa -in key.pem -pubout -out key.pub
/* 输出证书请求req.csr中的公钥部分 */
[root@xuexi ssl]# openssl req -noout -pubkey -in req.csr     
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC+YBneLYbh+OZWpiyPqIQHOsU5
D8il6UF7hi3NgEX/6vtciSmp7GXpXUV1tDglCCTPOfCHcEzeO0Gvky21LUenDsl/
aC2lraSijpl41+rT4mKNrCyDPZw4iG44+vLHfgHb3wJhBbBk0aw51dmxUat8FHCL
hU7nx+Du637UDlwdEQIDAQAB
-----END PUBLIC KEY-----
/* 查看key.pub，可以发现和req.csr中的公钥是一样的 */
[root@xuexi ssl]# cat key.pub 
-----BEGIN PUBLIC KEY-----
MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQC+YBneLYbh+OZWpiyPqIQHOsU5
D8il6UF7hi3NgEX/6vtciSmp7GXpXUV1tDglCCTPOfCHcEzeO0Gvky21LUenDsl/
aC2lraSijpl41+rT4mKNrCyDPZw4iG44+vLHfgHb3wJhBbBk0aw51dmxUat8FHCL
hU7nx+Du637UDlwdEQIDAQAB
-----END PUBLIC KEY-----
```

虽然创建证书请求时使用的是公钥，但是却不能使用-key选项指定公钥，而是只能指定私钥，因为req -new或-newkey选项会调用openssl rsa命令来提取公钥，指定公钥该调用将执行失败。

#### 使用x509伪命令创建CA

使用x509伪命令需要提供请求文件，因此需要先创建证书请求文件。由于x509伪命令签名时不读取配置文件，所以不需要设置配置文件，若需要某选项，只需使用x509中对应的选项来达成即可。

以下x509 -req用于自签名，需要-signkey提供签名所需私钥key.pem。

```
[root@xuexi ssl]# openssl req -new -keyout key.pem -out req.csr -config ssl.conf
[root@xuexi ssl]# openssl x509 -req -in req.csr -signkey key.pem -out x509.crt
```

#### 使用ca伪命令创建CA

使用ca伪命令自签名会读取配置文件中的ca部分，所以配置文件中所需的目录和文件结构都需要创建好，包括目录db、private、certs，文件db/index、db/serial，并向serial中写入一个序列号。由于是自签名，可以自行指定私钥文件，因此对于签名所需CA私钥文件无需放置在private目录中。

```
[root@xuexi ssl]# touch db/{serial,index}
[root@xuexi ssl]# echo "01" > db/serial
[root@xuexi ssl]# openssl req -new -keyout key.pem -out req.csr -config ssl.conf
[root@xuexi ssl]# openssl ca -selfsign -keyfile key.pem -in req.csr -config ssl.conf
```

在此签名过程中有两次询问，如下：

```
Certificate is to be certified until Nov 20 06:34:41 2026 GMT (3650 days)
Sign the certificate? [y/n]:y


1 out of 1 certificate requests certified, commit? [y/n]y
Write out database with 1 new entries
Data Base Updated
```

若要无交互，则使用-batch进入批处理模式。

```
[root@xuexi ssl]# openssl ca -selfsign -keyfile key.pem -in req.csr -config ssl.conf -batch
```

### 为其他证书请求签名

CA为其他请求或证书签名时，需要使用到的文件有：自己的CA证书和自己的私钥文件。因此签名过程中需要提供这两个文件。

#### 使用ca伪命令为其他证书请求签名

使用ca伪命令自建根CA后，目录结构如下：

```
[root@xuexi ssl]# tree -R -C             
.
├── certs
│   └── 01.pem
├── db
│   ├── index
│   ├── index.attr
│   ├── index.old
│   ├── serial
│   └── serial.old
├── key.pem
├── private
├── req.csr
└── ssl.conf
```

其中01.pem是根CA证书，key.pem是根CA私钥。

现在要为其他证书请求签名，首先创建其他请求吧，假设该请求文件/tmp/req.csr。

```
[root@xuexi ssl]# openssl req -new -keyout /tmp/key.pem -out /tmp/req.csr -config ssl.conf
```

使用根证书01.pem为/tmp/req.csr签名。

```
[root@xuexi ssl]# openssl ca -in /tmp/req.csr -keyfile key.pem -cert certs/01.pem -config ssl.conf -batch
```

这样挺麻烦，因为每次为别人签名时都要指定-cert和-keyfile，可以将CA的证书和CA的私钥移动到配置文件中指定的路径下：

```
certificate = $home/$name.crt
private_key = $home/private/$name.key
```



```
[root@xuexi ssl]# mv certs/01.pem root-ca.crt
[root@xuexi ssl]# mv key.pem private/root-ca.key
```

再使用ca签名时将可以使用默认值。

```
[root@xuexi ssl]# openssl ca -in /tmp/req.csr -config ssl.conf -batch
```

#### 使用x509伪命令为其他证书请求签名

现在根CA证书为root-ca.crt，CA的私钥为private/root-ca.key。

下面使用x509伪命令实现签名。由于x509不会读取配置文件，所以需要提供签名的序列号，使用-CAcreateserial可以在没有序列号文件时自动创建；由于x509默认-in指定的输入文件是证书文件，所以要对请求文件签名，需要使用-req来表示输入文件为请求文件。

```
[root@xuexi ssl]# openssl x509 -req -in /tmp/req.csr -CA root-ca.crt -CAkey private/root-ca.key -out x509.crt -CAcreateserial
```

## openssl证书签署和自签署：默认配置文件的实现方法

这是推荐采用的方法，因为方便管理，但使用默认配置文件，需要进行一些初始化动作。

由于完全采用/etc/pki/tls/openssl.cnf的配置，所以要建立相关文件。

自建CA的过程：

```
[root@xuexi tmp]# touch /etc/pki/CA/index.txt 
[root@xuexi tmp]# echo "01" > /etc/pki/CA/serial
# 创建CA的私钥
[root@xuexi tmp]# openssl genrsa -out /etc/pki/CA/private/cakey.pem
# 创建CA待自签署的证书请求文件
[root@xuexi tmp]# openssl req -new -key /etc/pki/CA/private/cakey.pem -out rootCA.csr
# 自签署
[root@xuexi tmp]# openssl ca -selfsign -in rootCA.csr
# 将自签署的证书按照配置文件的配置复制到指定位置
[root@xuexi tmp]# cp /etc/pki/CA/newcerts/01.pem /etc/pki/CA/cacert.pem
```

为他人颁发证书的过程：

```
[root@xuexi tmp]# openssl ca -in youwant1.csr
```

签署成功后，证书位于/etc/pki/CA/newcert目录下，将新生成的证书文件发送给申请者即可。



[1]: /linux/openssl_subcmds#blogpasswd	"openssl密码参数格式"

