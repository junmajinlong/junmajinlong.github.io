---
title: 加密、签名和SSL握手机制细节
p: linux/ssl_details.md
date: 2020-07-09 11:35:49
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# 加密、签名和SSL握手机制细节

## 背景知识

**对称加密**：加密解密使用同一密钥，加解密速度快。随着人数增多，密钥数量急增`n(n-1)/2`。

**非对称加密**：使用公私钥配对加解密，速度慢。公钥是从私钥中提取出来的，一般拿对方公钥加密来保证数据安全性，拿自己的私钥加密来证明数据来源的身份。

**单向加密**：不算是加密，也常称为散列运算，用于生成独一无二的校验码(或称为指纹、特征码)来保证数据的完整性和一致性，如MD5、SHA。具有雪崩效应，任何一点数据的改变，生成的校验码值变化非常大。

互联网数据安全可靠的条件：
- 1.数据来源可信，即数据发送者身份可信。
- 2.数据具备完整性，即数据未被修改过。
- 3.数据安全性，即数据不会被泄漏，他人截获后无法解密。

## 互联网数据加密的细节

对数据加密的方法有三种：对称加密、私钥加密和公钥加密。三种方法只靠其中任意一种都有不可容忍的缺点，因此考虑将它们结合使用。

考虑这三种加密算法的特性，公私钥加密速度慢，对称加密快，所以可以首先对数据部分使用对称加密。再进一步考虑，公钥大家都可以获取，若使用自己私钥加密，数据被截获后直接就被破解，因此使用对方的公钥加密，又由于公钥加密速度慢，所以可以使用对方公钥对对称密钥部分进行加密。

数据的收取者解密时，将使用自己的私钥解密第一层，得到对称密钥后加密的数据，再使用对称密钥解密，这样就能获得最终数据。

如下图所示分别是加密和解密的全过程。

![](E:\onedrive\docs\blog_imgs\733013-20161122152607940-1921693334.png)

加密的方法很多，但是上述方法是综合考虑互联网安全后较为成熟的一种简单加密方法。

使用上述方法**加密保证了数据的安全性**，但是还未保证数据的完整性、一致性以及数据来源的可靠性。

## 互联网数据签名的细节

在保证了数据的安全性后，还需要保证数据的完整性、一致性以及数据来源的可靠性。

对于**数据的完整性和一致性，使用单向加密算法**，通过hash函数计算出数据独一无二的校验码，**这个校验码称为【信息摘要(Message Digest)】**。

**对于数据来源可靠性，使用自己的私钥加密即可验证身份**，因为获得数据后使用公钥不能解密的就证明数据不是配对私钥加密的。但是私钥加密速度慢，所以只用私钥加密摘要信息，**加密后的摘要信息称为【数字签名(Signature)】**。

用户获得数字签名后的数据，首先使用数据来源方的公钥解密，这样获得了数据和信息摘要部分，并确认了数据来源的可靠性。由于这时候数据部分是没有被加密的，所以用户也可以使用同种单向加密算法计算出摘要信息，然后对比来源方的摘要信息和自己计算出的摘要信息，如果相等则证明数据完全未被修改过，是完整一致的。

**因此只要使用数字签名就能保证数据来源的可靠性、数据的完整性和一致性。**

如图所示分别是数字签名和确认数据的全过程。

![](E:\onedrive\docs\blog_imgs\733013-20161122152609003-1710587731.png)

## 互联网数据安全传输的细节

要在互联网上安全传输数据，要保证数据来源可靠、数据原始未被修改过、数据丢失不泄密。

如果数据传输双方张三和李四不在意数据丢失的泄露性，那么可以不对数据进行加密，只要数字签名即可。即，可以牺牲数据的安全性，只要保证数据的完整性、一致性和可靠性，即使被中间人王五截获了甚至截获后修改一番再发送给李四也无所谓，因为李四可以根据数字签名验证数据的来源及数据的完整性，若发现被修改后大不了不用了。现在互联网上很多时候下载软件时就提供了签名验证，使用的就是这种机制，不管软件是否被截取，只要安装者能验证即可，如下图。

![](E:\onedrive\docs\blog_imgs\733013-20170701104024539-1819774659.png)

但是如果在意数据泄漏呢？就需要将数字签名和加密结合起来使用。有两种方案：
- 1.先对数据加密，再对加密后的整体进行数字签名；
- 2.先对数据进行数字签名，再对签名后的整体进行加密。

在互联网上基本使用第二种方法，用户最终只对数据部分进行校验而不对加密后的数据进行校验。具体细节如下：首先进行数字签名，再使用对称加密加密签名后的整体，然后使用对方的公钥只加密对称密钥部分。这样即保证了加密速度，还保证了数据的安全性、可靠性和完整性。解密时反向进行即可。如图所示。

![](E:\onedrive\docs\blog_imgs\733013-20170701110248680-1763794011.png)

但是这时还有一个漏洞，问题出在数字签名过程中私钥加密以及后面公钥解密的不安全性。图中李四在拿公钥A解密的时候，这个公钥A真的是张三的公钥吗？也许张三传输公钥给李四的过程中被王五截断，王五声称自己是张三，并把自己的公钥给了李四，然后王五用自己的私钥对木马程序进行签名，进行对称加密后再使用李四的公钥加密，最后传输给李四，这样一来李四以为王五就是张三，导致的结果是李四对木马程序完全信任。

如何解决这个漏洞呢？只要保证李四获得的公钥A真的是来源于张三即可，如何保证呢？互联网之下，数据传输的两端可能谁都不认识谁，谁也不相信谁，所以最终还是依靠第三方组织——CA。

## CA、PKI及信任CA

CA(Certificate Authority)是数字证书认证中心，常称为证书颁发机构，申请者提交自己的**公钥**和一些个人信息(如申请者国家，姓名，单位等)给CA，CA对申请者的这些信息单向加密生成摘要信息，然后使用**自己的私钥**加密整个摘要信息，这样就得到了CA对申请者的数字签名，在数字签名上再加上CA自己的一些信息(如CA的机构名称，CA层次路径等)以及该证书的信息(如证书有效期限)，就得到了所谓的数字证书。

过程如下图。

![](E:\onedrive\docs\blog_imgs\733013-20170703215702925-2009564655.png)

如果某用户信任了该CA，就获取了该CA的公钥（实际上信任CA的其中一个作用就是获取CA公钥），使用该公钥解密数字证书就可以验证申请者的信息以及申请者公钥的可靠性（申请者的公钥只被CA的私钥加密，解密该私钥后只是需要验证可靠性）。

这里的关键是CA使用自己的私钥给申请者加密，那么如何保证CA是可信并且合法的呢？根CA是通过自签署数字证书的方式标榜自己的可信性和合法性，第一级子CA由根CA颁发合法数字证书，第二级直至所有的子CA都由上一级子CA颁发数字证书。对于多级子CA只需要信任根CA即可，因为获取了根CA的公钥，可以解密第一级子CA的证书并获取验证第一级子CA的公钥，层层递进，最终获取到为申请者颁发数字证书的机构并获取它的公钥。

另一种情况，虽然信任了根CA，但如果根CA授权的某个二级或者某个二级授权的三级CA不在信任列表中，就需要将这个中间的CA补齐，否则中间出现断层，无法成功解密该中间CA颁发的证书。通常这种中间CA的证书称为chain证书，例如`xxx.com.chain.crt`。如果购买的证书发给你的文件中包含了chain.crt文件，则说明他是中间CA，且有些浏览器或操作系统中没有内置它的CA，这时应该将chain证书也配置到web server上。

正是这些根CA和子CA组成了PKI。

信任CA后，每次接收到需要解密的数字证书时，还要去该颁发机构指定网站的证书吊销列表（CRL）中查询该证书是否被吊销，对于吊销后的证书应该不予以信任，这是信任CA的第二个作用。导致证书被吊销的可能性不少，例如申请者的私钥被黑客获取，申请者申请吊销等。

也有公司使用自签的证书，例如某些银行、12306有时候就要求下载证书并安装。使用自签证书的好处当然是省钱、方便啦。

## 数字证书类型和内容

PKI的两种实现方式TLS和SSL使用的证书格式都是x509，TLSv1和SSLv3基本等价，只不过SSL实现在OSI 4层模型中的应用层和传输层的中间，TLS实现在传输层。

还有PKI的另一种实现方式GPG，它的证书使用的不是x509格式。

数字证书中包含的信息有：申请者的公钥，证书有效期，证书合法拥有人，证书如何被使用，CA的信息，CA对申请者信息的数字签名。

![](E:\onedrive\docs\blog_imgs\733013-20161122152612643-290583211.png)

![](E:\onedrive\docs\blog_imgs\733013-20161122152613706-1069153031.png)

![](E:\onedrive\docs\blog_imgs\733013-20161122152615190-733355526.png)

## SSL握手机制

有了CA颁发的数字证书后，通信机制就和下图的机制完全不同了。

![img](https://images2015.cnblogs.com/blog/733013/201707/733013-20170701110248680-1763794011.png)

上图中每一段数据都签名加密，有了数字证书后实际上已经验证了身份，不需要每一段数据都签名，这能提升效率。在上图中的漏洞是无法确认获取的公钥A是否可信，有了数字证书后已经能够确认公钥A是可信的。但问题是公钥A本来目的是用来解密数字签名的，有了数字证书后不需要数字签名了，那公钥A不是多余的吗，如果多余，那把公钥A交给CA是不是也是多余的呢？

不多余，因为SSL的握手机制和数字签名机制完全不同。以下是单向验证机制，只验证服务端。

![](E:\onedrive\docs\blog_imgs\733013-20161122152620034-469460891.png)

**第一步**：Visitor给出协议版本号、一个客户端随机数（Client random），以及客户端支持的加密方法。

**第二步**：Server确认双方使用的加密方法，以及一个服务器生成的随机数（Server random）。

**第三步**：Server发送数字证书给Visitor。

**第四步**：Visitor确认数字证书有效（查看证书状态且查询证书吊销列表），并使用信任的CA的公钥解密数字证书获得Server的公钥，然后生成一个新的46字节随机数（称为预备主密钥Pre-master secret），并使用Server的公钥加密预备主密钥发给Server。

**第五步**：Server使用自己的私钥，解密Visitor发来的预备主密钥。

**第六步**：Visitor和Server双方都具有了(客户端随机数+服务端随机数+预备主密钥)，它们两者都根据约定的加密方法，使用这三个随机数生成对称密钥——主密钥（也称为对话密钥session key），用来加密接下来的整个对话过程。

**第七步**：在双方验证完session key的有效性之后，SSL握手机制就算结束了。之后所有的数据只需要使用“对话密钥”加密即可，不再需要多余的加密机制。

需要说明的是，session key不是真正的对称加密密钥，而是由session key进行hash算法得到一段hash值，从这个hash值中推断出对称加密过程中所需的key(即对称加密所需的明文密码部分)、salt(在RFC文档中称为MAC secret)和IV向量。以后客户端每次传输数据，都需要用key + salt + IV向量来完成对称加密，而服务端只需一个key和协商好的加密算法即可解密。同理服务端向客户端传输数据时也是一样的。

> 注意：
> 1.在SSL握手机制中，需要三个随机数（客户端随机数+服务端随机数+预备主密钥）；
> 2.至始至终客户端和服务端只有一次非对称加密动作——客户端使用证书中获得的服务端公钥加密预备主密钥。
> 3.上述SSL握手机制的前提单向验证，无需验证客户端，如果需要验证客户端则可能需要客户端的证书或客户端提供签名等。


Server和Visitor通信，Server把数字证书发给Visitor，最关键的一点是Visitor要保证证书的有效性，通过查看证书状态并去CA的吊销列表查看Server的证书是否被吊销。只有Server的证书可用了，才保证了第一环节的安全性。

可以看出，使用SSL比前文介绍的“数字签名+加密”简便多了，将身份验证和密钥生成在会话的开始就完成了，而不需要每次的数据传输过程中都进行，这就是**https等使用ssl加密机制的握手通信过程**。

 

关于https的握手协议过程RFC文档：<https://tools.ietf.org/html/rfc6101#appendix-F.1>。