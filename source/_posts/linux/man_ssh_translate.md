---
title: ssh命令中文手册(man ssh翻译)
p: linux/man_ssh_translate.md
date: 2020-04-13 18:20:30
tags: Linux
categories: Linux
---

--------

**[回到Linux基础系列文章大纲](/linux/index)**  
**[回到Shell系列文章大纲](/shell/index)**  

--------

# ssh命令中文手册(man ssh翻译)

```
SSH(1)                    BSD General Commands Manual                   SSH(1)

NAME
     ssh — OpenSSH SSH 客户端工具(远程登录程序)

SYNOPSIS
     ssh [-1246AaCfGgKkMNnqsTtVvXxYy] [-b bind_address] [-c cipher_spec]
         [-D [bind_address:]port] [-E log_file] [-e escape_char]
         [-F configfile] [-I pkcs11] [-i identity_file] [-L address]
         [-l login_name] [-m mac_spec] [-O ctl_cmd] [-o option] [-p port]
         [-Q query_option] [-R address] [-S ctl_path] [-W host:port]
         [-w local_tun[:remote_tun]] [user@]hostname [command]

DESCRIPTION
     ssh(SSH客户端)是一个登陆远程主机和在远程主机上执行命令的程序。它的目
     的是在不安全的网络中为两个互不信任的主机提供安全加密的通信方式。也
     可以通过安全隧道被转发X11连接、任意TCP端口和UNIX套接字上的数据包。
     
     ssh连接并登录指定的主机(还可以指定用户名)。客户端必须提供身份标识给
     远程主机，提供方式有多种，见下文。
     
     如果ssh命令行中指定了命令，则将在远程主机上执行而不是登录远程主机。
    
     选项说明如下：
    
     -1      强制使用ssh v1版本。
    
     -2      强制使用ssh v2版本。
    
     -4      强制只使用IPv4地址。
    
     -6      强制只使用IPv6地址。
    
     -A      启用代理转发功能，也可在全局配置文件(/etc/ssh/config)中配置。
             代理转发功能应该要谨慎开启。
    
     -a      禁用代理转发功能。
    
     -b bind_address  
             在本地主机上绑定用于ssh连接的地址，当系统有多个ip时才生效。
    
     -C      请求会话间的数据压缩传递。对于网络缓慢的主机，压缩对连接有所
             提升。但对网络流畅的主机来说，压缩只会更糟糕。
             
     -c      选择ssh会话间数据加密算法。
    
     -D [bind_address:]port
             指定一个本地动态应用层端口做转发端口。工作方式是分配一个套接
             字监听在此端口，当监听到此端口有连接时，此连接中的数据将通过
             安全隧道转发到server端，server端再和目的地(端口)建立连接，目
             的地(端口)由应用层协议决定。目前支SOCK4和SOCK5两种协议，并且
             SSH将扮演SOCKS服务端角色。
             
             只有root用户可以开启特权端口。动态转发端口也可以在配置文件
             中指定。
             
             默认情况下，转发端口将绑定在GatewayPorts指令指定的地址上，但
             是可以显式指定bind_address，如果bind_address设置为"localhost"，
             则转发端口将绑定在回环地址上，如果bind_address不设置或设置为
             "*"，则转发端口绑定在所有网路接口上。
    
     -E log_file
             将debug日志写入到log_file中，而不是默认的标准错误输出stderr。
    
     -e escape_char 
             设置逃逸首字符，默认为"~"，设置为"none"将禁用逃逸字符，并使
             得会话完全透明。详细用法见后文。
    
     -F configfile 
             指定用户配置文件，默认为~/.ssh/config，如果在命令行指定了该
             选项，则全局配置文件/etc/ssh_config将被忽略。
    
     -f      请求ssh在工作在后台模式。该选项隐含了"-n"选项，所以标准输入
             将变为/dev/null。
    
     -G      使用该选项将使得ssh在匹配完Host后将输出与之对应的配置选项，
             然后退出
    
     -g      允许远程主机连接到本地转发端口上。
    
     -I pkcs11
             Specify the PKCS#11 shared library ssh should use to communicate
             with a PKCS#11 token providing the user's private RSA key.
    
     -i identity_file  
             指定公钥认证时要读取的私钥文件。默认为~/.ssh/id_rsa。
    
     -K      启用GSSAPI认证并将GSSAPI凭据转发(分派)到服务端。
    
     -k      禁止转发(分派)GSSAPI凭据到服务端。
    
     -L [bind_address:]port:host:hostport
     -L [bind_address:]port:remote_socket
     -L local_socket:host:hostport
     -L local_socket:remote_socket
             对本地指定的TCP端口port的连接都将转发到指定的远程主机及其端
             口上(host:hostport)。工作方式是在本地端分配一个socket监听TCP
             端口。当监听到本地此端口有连接时，连接将通过安全隧道转发给
             远程主机(server)，然后从远程主机(是server端)上建立一个到
             host:hostport的连接，完成数据转发。
             
             译者注：隧道建立在本地和远程主机(server端，即中间主机)之间，
             而非本地和host之间，也不是远程主机和host之间。
    
             端口转发也可以在配置文件中指定。只有root用户才能转发特权端口
             (小于1024)。
    
             默认本地端口被绑定在GatewayPorts指令指定的地址上。但是，显式
             指定的bind_address可以用于绑定连接到指定的地址上。如果设置
             bind_address为"localhost"，则表示被绑定的监听端口只可用于本地
             连接(即该端口监听在回环地址上)，如果不设置bind_address或设置
             为"*"则表示绑定的端口可用于所有网络接口上的连接(即表示该端口
             监听在所有地址上)。
    
     -l login_name 
             指定登录在远程机器上的用户名。这也可以在全局配置文件中设置。
    
     -M      将ssh客户端置入"master"模式，以便连接共享(连接复用)。
             即实现ControlMaster和ControlPersist的相关功能。
    
     -m mac_spec
             A comma-separated list of MAC (message authentication code)
             algorithms, specified in order of preference.  See the MACs key‐
             word for more information.
    
     -N      明确表示不执行远程命令。仅作端口转发时比较有用。
    
     -n      将/dev/null作为标准输入stdin，可以防止从标准输入中读取内容。
             当ssh在后台运行时必须使用该项。但当ssh被询问输入密码时失效。


     -O ctl_cmd
             Control an active connection multiplexing master process.  When
             the -O option is specified, the ctl_cmd argument is interpreted
             and passed to the master process.  Valid commands are: “check”
             (check that the master process is running), “forward” (request
             forwardings without command execution), “cancel” (cancel for‐
             wardings), “exit” (request the master to exit), and “stop”
             (request the master to stop accepting further multiplexing
             requests).
    
     -o option
             Can be used to give options in the format used in the configura‐
             tion file.  This is useful for specifying options for which there
             is no separate command-line flag.  For full details of the
             options listed below, and their possible values, see
             ssh_config(5).
    
                   AddKeysToAgent
                   AddressFamily
                   BatchMode
                   BindAddress
                   CanonicalDomains
                   CanonicalizeFallbackLocal
                   CanonicalizeHostname
                   CanonicalizeMaxDots
                   CanonicalizePermittedCNAMEs
                   CertificateFile
                   ChallengeResponseAuthentication
                   CheckHostIP
                   Cipher
                   Ciphers
                   ClearAllForwardings
                   Compression
                   CompressionLevel
                   ConnectionAttempts
                   ConnectTimeout
                   ControlMaster
                   ControlPath
                   ControlPersist
                   DynamicForward
                   EscapeChar
                   ExitOnForwardFailure
                   FingerprintHash
                   ForwardAgent
                   ForwardX11
                   ForwardX11Timeout
                   ForwardX11Trusted
                   GatewayPorts
                   GlobalKnownHostsFile
                   GSSAPIAuthentication
                   GSSAPIDelegateCredentials
                   HashKnownHosts
                   Host
                   HostbasedAuthentication
                   HostbasedKeyTypes
                   HostKeyAlgorithms
                   HostKeyAlias
                   HostName
                   IdentityFile
                   IdentitiesOnly
                   IPQoS
                   KbdInteractiveAuthentication
                   KbdInteractiveDevices
                   KexAlgorithms
                   LocalCommand
                   LocalForward
                   LogLevel
                   MACs
                   Match
                   NoHostAuthenticationForLocalhost
                   NumberOfPasswordPrompts
                   PasswordAuthentication
                   PermitLocalCommand
                   PKCS11Provider
                   Port
                   PreferredAuthentications
                   Protocol
                   ProxyCommand
                   ProxyUseFdpass
                   PubkeyAcceptedKeyTypes
                   PubkeyAuthentication
                   RekeyLimit
                   RemoteForward
                   RequestTTY
                   RhostsRSAAuthentication
                   RSAAuthentication
                   SendEnv
                   ServerAliveInterval
                   ServerAliveCountMax
                   StreamLocalBindMask
                   StreamLocalBindUnlink
                   StrictHostKeyChecking
                   TCPKeepAlive
                   Tunnel
                   TunnelDevice
                   UpdateHostKeys
                   UsePrivilegedPort
                   User
                   UserKnownHostsFile
                   VerifyHostKeyDNS
                   VisualHostKey
                   XAuthLocation
    
     -p port 
             指定要连接远程主机上哪个端口，也可在全局配置文件中指定。
    
     -Q query_option
             Queries ssh for the algorithms supported for the specified ver‐
             sion 2.  The available features are: cipher (supported symmetric
             ciphers), cipher-auth (supported symmetric ciphers that support
             authenticated encryption), mac (supported message integrity
             codes), kex (key exchange algorithms), key (key types), key-cert
             (certificate key types), key-plain (non-certificate key types),
             and protocol-version (supported SSH protocol versions).
    
     -q      静默模式。大多数警告信息将不输出。
    
     -R [bind_address:]port:host:hostport
     -R [bind_address:]port:local_socket
     -R remote_socket:host:hostport
     -R remote_socket:local_socket
             对远程(server端)指定的TCP端口port的连接都就将转发到本地主机和
             端口上，工作方式是在远端(server)分配一个套接字socket监听TCP端
             口。当监听到此端口有连接时，连接将通过安全隧道转发给本地，然后
            从本地主机建一条到host:hostport的连接。
             端口转发也可以在配置文件中指定。只有root用户才能转发特权端口
             (小于1024)。
             默认远程(server)套接字被绑定在回环地址上。但是，显式指定的
             bind_address可以用于绑定套接字到指定的地址上。如果不设置
             bind_address或设置为"*"则表示套接字监听在所有网络接口上。
             只有当远程(server)主机的GatewayPorts选项开启时，指定的
             bind_address才能生效。(见sshd_config(5))。
             如果port值为0，远程主机(server)监听的端口将被动态分配，并且在
             运行时报告给客户端。
    
     -S ctl_path
             Specifies the location of a control socket for connection shar‐
             ing, or the string “none” to disable connection sharing.  Refer
             to the description of ControlPath and ControlMaster in
             ssh_config(5) for details.
    
     -s      请求在远程主机上调用一个子系统(subsystem)。子系统有助于ssh为
             其他程序(如sftp)提供安全传输。子系统由远程命令指定。
    
     -T      禁止为ssh分配伪终端。
    
     -t      强制分配伪终端，重复使用该选项"-tt"将进一步强制。
    
     -V      显示版本号并退出。
    
     -v      详细模式，将输出debug消息，可用于调试。"-vvv"可更详细。
    
     -W host:port
             请求客户端上的标准输入和输出通过安全隧道转发到host:port上，该选
             项隐含了"-N","-T",ExitOnForwardFailure和ClearAllForwardings选项。
    
     -w local_tun[:remote_tun]
             Requests tunnel device forwarding with the specified tun(4)
             devices between the client (local_tun) and the server
             (remote_tun).
    
             The devices may be specified by numerical ID or the keyword
             “any”, which uses the next available tunnel device.  If
             remote_tun is not specified, it defaults to “any”.  See also
             the Tunnel and TunnelDevice directives in ssh_config(5).  If the
             Tunnel directive is unset, it is set to the default tunnel mode,
             which is “point-to-point”.
    
     -X      Enables X11 forwarding.  This can also be specified on a per-host
             basis in a configuration file.
    
             X11 forwarding should be enabled with caution.  Users with the
             ability to bypass file permissions on the remote host (for the
             user's X authorization database) can access the local X11 display
             through the forwarded connection.  An attacker may then be able
             to perform activities such as keystroke monitoring.
    
             For this reason, X11 forwarding is subjected to X11 SECURITY
             extension restrictions by default.  Please refer to the ssh -Y
             option and the ForwardX11Trusted directive in ssh_config(5) for
             more information.
             
     -x      Disables X11 forwarding.
     -Y      Enables trusted X11 forwarding.  Trusted X11 forwardings are not
             subjected to the X11 SECURITY extension controls.
    
     -y      使用syslog发送日志信息。默认情况下日志信息发送到标准错误输出
    
     除了从命令行获取配置信息，还可以从用户配置文件和全局配置文件中
     获取额外配置信息。详细信息见ssh_config(5)

认证机制
     可用的认证机制及它们的先后顺序为：GSSAPI-based,host-based,public key,
     challenge-response,password。PreferredAuthentications选项可以改变默认的认证顺序 
         
     Host-based authentication works as follows: If the machine the user logs
     in from is listed in /etc/hosts.equiv or /etc/shosts.equiv on the remote
     machine, and the user names are the same on both sides, or if the files
     ~/.rhosts or ~/.shosts exist in the user's home directory on the remote
     machine and contain a line containing the name of the client machine and
     the name of the user on that machine, the user is considered for login.
     Additionally, the server must be able to verify the client's host key
     (see the description of /etc/ssh_known_hosts and ~/.ssh/known_hosts,
     below) for login to be permitted.  This authentication method closes
     security holes due to IP spoofing, DNS spoofing, and routing spoofing.
     [Note to the administrator: /etc/hosts.equiv, ~/.rhosts, and the
     rlogin/rsh protocol in general, are inherently insecure and should be
     disabled if security is desired.]
    
     公钥认证机制：用户创建公钥/私钥密钥对，将公钥发送给服务端，所以服务端
     知道的是公钥，私钥只有自己知道。    
    
     ~/.ssh/authorized_keys文件列出了允许登录的公钥。当用户登录时，ssh客户端程序
     告诉服务端程序要使用哪个密钥对来完成身份验证。客户端提供自己要使用的私钥，
     而服务端则检查对应的公钥以确定是否要接受该客户端的登录。
    
     用户使用ssh-keygen创建密钥对(以rsa算法为例)，将保存在~/.ssh/id_rsa和~/.ssh/id_rsa.pub。
     然后该用户拷贝公钥文件到远程主机上某用户(如A)家目录下的~/.ssh/authorized_keys，
     之后用户就可以以用户A的身份登录到远程主机上。
    
     公钥认证机制的一种变体是证书认证：只有被信任的证书才允许连接。
     详细信息见ssh-keygen(1)的CERTIFICATES段说明。
    
     使用公钥认证机制或证书认证机制最方便的方法是"认证代理"，
     详细信息见ssh-agent(1)和ssh_config(5)中的AddKeysToAgent指令段。
    
     Challenge-response authentication works as follows: The server sends an
     arbitrary "challenge" text, and prompts for a response.  Examples of
     challenge-response authentication include BSD Authentication (see
     login.conf(5)) and PAM (some non-OpenBSD systems).
    
     最后，如果所有认证方法都失败，将提示输入密码。输入的密码将被加密传送，
     然后被服务端检测是否正确。
    
     SSH客户端自动维护和检查一个主机认证信息数据库，所有已知的主机公钥都会
     记录到此文件中。主机信息条目(host key)存放在~/.ssh/known_hosts文件中。
     另外，在检查host key时，/etc/ssh_known_hosts也会被自动检测。
     当host key被改变时，ssh将发出警告，并禁止密钥认证机制以防止服务端欺骗
     或中间人攻击。选项StrictHostKeyChecking选项可用于控制登录时那些未知host key
     如何处理。
    
     当客户端被服务端接受，服务端将以非交互会话执行给定的命令，若没有给定命令，
     则登录到服务端，并进入到交互会话模式，同时会为登录的用户分配shell，之后
     所有的交互信息都将被加密传输。
     
     ssh默认会请求交互式会话，这将请求一个伪终端(pty)，使用"-T"或"-t"选项可以
     改变该行为，"-T"是禁止分配伪终端，"-t"则是强制分配伪终端，可使用"-tt"
     表示进一步强制。
    
     如果为ssh分配了伪终端，则用户可以在此伪终端中使用逃逸字符实现特殊控制。
    
     如果未分配伪终端给ssh，则连接会话是透明的，可以用来可靠传输二进制数据。
     如果设置逃逸字符为"none"，将使得会话透明，即使它使用了tty终端。
    
     当命令结束或shell退出时将终止会话连接，所有的X11和TCP连接也都被关闭。

逃逸字符
     当分配了伪终端时，ssh支持一系列的逃逸字符实现特殊功能。

     默认的逃逸首字符为"~"，其后可跟某些特定字符(如下列出)，逃逸字符必须放
     在行尾以实现特定的中断。可在配置文件中使用EscapeChar指令或命令行的"-e"
     选项来改变逃逸首字符。
    
     ~.      禁止连接
    
     ~^Z     将ssh放入后台
    
     ~#      列出已转发的连接
    
     ~&      Background ssh at logout when waiting for forwarded connection /
             X11 sessions to terminate.
    
     ~?      列出逃逸字符列表
    
     ~B      发送BREAK信号给远程主机
    
     ~C      打开命令行。Open command line.  Currently this allows the addition of port
             forwardings using the -L, -R and -D options (see above).  It also
             allows the cancellation of existing port-forwardings with
             -KL[bind_address:]port for local, -KR[bind_address:]port for
             remote and -KD[bind_address:]port for dynamic port-forwardings.
             !command allows the user to execute a local command if the
             PermitLocalCommand option is enabled in ssh_config(5).  Basic
             help is available, using the -h option.
    
     ~R      请求该会话进行密钥更新
    
     ~V      当错误被写入到stderr时，降低信息的详细程度(loglevel)
    
     ~v      当错误被写入到stderr时，增加信息的详细程度

TCP转发
     可在配置文件或命令行选项上开启基于安全隧道的任意TCP连接转发功能。
     一个TCP转发可能的应用场景是为了安全连接到邮件服务器，其他场景则主要
     是为了穿过防火墙。

     下面的例子中，建立了IRC客户端和服务端的加密连接，尽管IRC服务端不直
     接支持加密连接。用户在本地指定一个用于转发到远程服务器上的端口，这
     样在本地主机上将开启一个加密的服务，当连接到本地转发端口时，ssh将
     加密和转发此连接。   
    
     下面的示例中，从客户端主机"127.0.0.1"到"server.example.com"的连接将
     使用隧道技术。
    
         $ ssh -f -L 1234:localhost:6667 server.example.com sleep 10
         $ irc -c '#users' -p 1234 pinky 127.0.0.1
    
     这个隧道建立在本地和"server.example.com"之间，隧道传递的内容有:
     “#users","pinky",using port 1234. 无论使用的是什么端口，只要大于
     1023(只有root可以在特权端口上建立套接字)，即使端口已被使用也不
     会发生冲突。连接将被转发到远程主机的6667端口上，因为IRC服务的
     默认端口为6667。 
    
     "-f"选项将ssh放入后台，而远程命令"sleep 10"则表示在一段时间(10秒)
     内的连接将通过隧道传输。如果在10秒内没有连接，则ssh退出。
     (也就是说该隧道只在后台保持10秒钟。)

X11 FORWARDING
     If the ForwardX11 variable is set to “yes” (or see the description of
     the -X, -x, and -Y options above) and the user is using X11 (the DISPLAY
     environment variable is set), the connection to the X11 display is auto‐
     matically forwarded to the remote side in such a way that any X11 pro‐
     grams started from the shell (or command) will go through the encrypted
     channel, and the connection to the real X server will be made from the
     local machine.  The user should not manually set DISPLAY.  Forwarding of
     X11 connections can be configured on the command line or in configuration
     files.

     The DISPLAY value set by ssh will point to the server machine, but with a
     display number greater than zero.  This is normal, and happens because
     ssh creates a “proxy” X server on the server machine for forwarding the
     connections over the encrypted channel.
    
     ssh will also automatically set up Xauthority data on the server machine.
     For this purpose, it will generate a random authorization cookie, store
     it in Xauthority on the server, and verify that any forwarded connections
     carry this cookie and replace it by the real cookie when the connection
     is opened.  The real authentication cookie is never sent to the server
     machine (and no cookies are sent in the plain).
    
     If the ForwardAgent variable is set to “yes” (or see the description of
     the -A and -a options above) and the user is using an authentication
     agent, the connection to the agent is automatically forwarded to the
     remote side.

VERIFYING HOST KEYS
     当用户第一次连接到一个服务端，将输出服务端公钥的指纹(fingerprint)给用户
     (除非StrictHostKeyChecking配置被禁用了)。这些指纹可通过ssh-keygen来计算。

           $ ssh-keygen -l -f /etc/ssh/ssh_host_rsa_key
    
     如果某指纹已经存在，可决定对应的密钥是接受还是拒绝。如果仅能获取到服
     务端的传统指纹(MD5)，ssh-keygen的"-E"选项可能会将指纹降级以做指纹匹配。
    
     由于仅通过查找指纹来比较host key比较困难，所以也支持使用随机数的方式
     可视化比较host key。通过设置VisualHostKey选项为"yes"，客户端连接服务
     端时将显示一小段ASCII图形信息(即图形化的指纹)，无论会话是否是需要交互
     的。通过比较已生成的图形指纹，用户可以轻松地找出host key是否发生了改
     变。但是，由于图形指纹不是很明了，所以相似的图形指纹并不能保证host key
     是没有改变过的，只不过通过图形指纹的方式提供了一个比较好的比较方式。


     要获取所有已知主机(known host)的图形指纹列表，使用下面的命令：
    
           $ ssh-keygen -lv -f ~/.ssh/known_hosts
    
     如果指纹是未知的，有一种方法可以验证它：使用DNS。可在DNS的区域文件中添
     加资源记录SSHFP，这样客户端就可以匹配那些已存在的主机指纹。
     
     在下面的例子中，将使用客户端连接到服务端"host.example.com"。但在此之前，
     应该先将"host.example.com"的SSHFP资源记录添加到DNS区域文件中：
    
           $ ssh-keygen -r host.example.com.
    
     将上面命令的输出结果添加到区域文件中。可以检查该资源记录是否可解析：
    
           $ dig -t SSHFP host.example.com
    
     最后使用客户端去连接服务端:
    
           $ ssh -o "VerifyHostKeyDNS ask" host.example.com
           [...]
           Matching host key fingerprint found in DNS.
           Are you sure you want to continue connecting (yes/no)?
    
     更多信息请查看ssh_config(5)的VerifyHostKeyDNS选项说明段。

SSH-BASED VIRTUAL PRIVATE NETWORKS(基于SSH的VPN)
     ssh可以通过网络伪设备tun(4)来支持Virtual Private Network(VPN)，使得
     两个网络之间可以安全通信。配置文件sshd_config(5)中的PermitTunnel选
     项控制是否在服务端支持VPN隧道，并控制在2层还是3层网络上启用。

     The following example would connect client network 10.0.50.0/24 with
     remote network 10.0.99.0/24 using a point-to-point connection from
     10.1.1.1 to 10.1.1.2, provided that the SSH server running on the gateway
     to the remote network, at 192.168.1.15, allows it.
    
     在客户端上:
    
           # ssh -f -w 0:1 192.168.1.15 true
           # ifconfig tun0 10.1.1.1 10.1.1.2 netmask 255.255.255.252
           # route add 10.0.99.0/24 10.1.1.2
    
     在服务端上:
    
           # ifconfig tun1 10.1.1.2 10.1.1.1 netmask 255.255.255.252
           # route add 10.0.50.0/24 10.1.1.1
    
     Client access may be more finely tuned via the /root/.ssh/authorized_keys
     file (see below) and the PermitRootLogin server option.  The following
     entry would permit connections on tun(4) device 1 from user “jane” and
     on tun device 2 from user “john”, if PermitRootLogin is set to
     “forced-commands-only”:
    
       tunnel="1",command="sh /etc/netstart tun1" ssh-rsa ... jane
       tunnel="2",command="sh /etc/netstart tun2" ssh-rsa ... john
    
     Since an SSH-based setup entails a fair amount of overhead, it may be
     more suited to temporary setups, such as for wireless VPNs.  More perma‐
     nent VPNs are better provided by tools such as ipsecctl(8) and
     isakmpd(8).

ENVIRONMENT
     ssh will normally set the following environment variables:

     DISPLAY               The DISPLAY variable indicates the location of the
                           X11 server.  It is automatically set by ssh to
                           point to a value of the form “hostname:n”, where
                           “hostname” indicates the host where the shell
                           runs, and ‘n’ is an integer ≥ 1.  ssh uses this
                           special value to forward X11 connections over the
                           secure channel.  The user should normally not set
                           DISPLAY explicitly, as that will render the X11
                           connection insecure (and will require the user to
                           manually copy any required authorization cookies).
    
     HOME                  Set to the path of the user's home directory.
    
     LOGNAME               Synonym for USER; set for compatibility with sys‐
                           tems that use this variable.
    
     MAIL                  Set to the path of the user's mailbox.
    
     PATH                  Set to the default PATH, as specified when compil‐
                           ing ssh.
    
     SSH_ASKPASS           If ssh needs a passphrase, it will read the
                           passphrase from the current terminal if it was run
                           from a terminal.  If ssh does not have a terminal
                           associated with it but DISPLAY and SSH_ASKPASS are
                           set, it will execute the program specified by
                           SSH_ASKPASS and open an X11 window to read the
                           passphrase.  This is particularly useful when
                           calling ssh from a .xsession or related script.
                           (Note that on some machines it may be necessary to
                           redirect the input from /dev/null to make this
                           work.)
    
     SSH_AUTH_SOCK         Identifies the path of a UNIX-domain socket used to
                           communicate with the agent.
    
     SSH_CONNECTION        Identifies the client and server ends of the con‐
                           nection.  The variable contains four space-sepa‐
                           rated values: client IP address, client port num‐
                           ber, server IP address, and server port number.
    
     SSH_ORIGINAL_COMMAND  This variable contains the original command line if
                           a forced command is executed.  It can be used to
                           extract the original arguments.
    
     SSH_TTY               This is set to the name of the tty (path to the
                           device) associated with the current shell or com‐
                           mand.  If the current session has no tty, this
                           variable is not set.
    
     TZ                    This variable is set to indicate the present time
                           zone if it was set when the daemon was started
                           (i.e. the daemon passes the value on to new con‐
                           nections).
    
     USER                  Set to the name of the user logging in.
    
     Additionally, ssh reads ~/.ssh/environment, and adds lines of the format
     “VARNAME=value” to the environment if the file exists and users are
     allowed to change their environment.  For more information, see the
     PermitUserEnvironment option in sshd_config(5).

FILES
     ~/.rhosts
             这个文件用于基于主机的认证机制(见上文)，里面列出允许登录的
             主机/用户对。该文件属主必须是这个对应的用户，且其它用户不
             能有写权限。但如果用户家目录位于NFS分区上时，该文件要求全
             局可读，因为sshd(8)使用root身份读取该文件。大多数情况下，
             推荐权限为"600"。

     ~/.shosts
             该文件的用法与".rhosts"完全一样，但允许基于主机认证的同时
             禁止使用"rlogin/rsh"登录。
    
     ~/.ssh/
             该目录是所有用户配置文件和用户认证信息的默认放置目录。虽然
             没有规定要保证该目录中内容的安全，但推荐其内文件只对所有者
             有读/写/执行权限，对其他人完全拒绝。
    
     ~/.ssh/authorized_keys
             该文件列出了可以用来登录的用户的公钥(DSA,ECDSA,Ed25519,RSA)。
             在sshd(8)的man文档中描述了该文件的格式。该文件不需要高安全性，
             但推荐只有其所有者有读/写权限，对其他人完全拒绝。
    
     ~/.ssh/config
             该文件是ssh的用户配置文件。在ssh_config(5)的man文档中描述了该
             文件的格式。由于可能会滥用该文件，该文件有严格的权限要求：只
             对所有者有读/写权限，对其他人完全拒绝写权限。
    
     ~/.ssh/environment
             包含了额外定义的环境变量。见上文ENVIRONMENT。
    
     ~/.ssh/identity
     ~/.ssh/id_dsa
     ~/.ssh/id_ecdsa
     ~/.ssh/id_ed25519
     ~/.ssh/id_rsa
             包含了认证的私钥。这些文件包含了敏感数据，应该只对所有者可读，
             并拒绝其他人的所有权限(rwx)。如果该文件可被其他人访问，则ssh
             会忽略该文件。可以在生产密钥文件的时候指定passphrase使用3DES
             算法加密该文件。
    
     ~/.ssh/identity.pub
     ~/.ssh/id_dsa.pub
     ~/.ssh/id_ecdsa.pub
     ~/.ssh/id_ed25519.pub
     ~/.ssh/id_rsa.pub
             包含了认证时的公钥。这些文件中的数据不敏感，允许任何人读取。
    
     ~/.ssh/known_hosts
             包含了所有已知主机的host key列表。该文件的详细格式见sshd(8)。
    
     ~/.ssh/rc
             该文件包含了用户使用ssh登录成功，但启用shell(或指定命令执行)
             之前执行的命令。详细信息见sshd(8)的man文档。
             (译者注：也就是说，登录成功后做的第一件事就是执行该文件中的
             命令)
    
     /etc/ssh/hosts.equiv
             该文件是基于主机认证的文件(见上文)。应该只能让root有写权限。
    
     /etc/ssh/shosts.equiv
             用法等同于"hosts.equiv"，但允许基于主机认证的同时禁止使用
             "rlogin/rsh"登录。
    
     /etc/ssh/ssh_config
             ssh的全局配置文件。该文件的格式和选项信息见ssh_config(5)。
    
     /etc/ssh/ssh_host_key
     /etc/ssh/ssh_host_dsa_key
     /etc/ssh/ssh_host_ecdsa_key
     /etc/ssh/ssh_host_ed25519_key
     /etc/ssh/ssh_host_rsa_key
             这些文件包含了host key的私密部分信息，它们用于基于主机认证。
    
     /etc/ssh/ssh_known_hosts
             已知host key的全局列表文件。该文件中要包含的host key应该由
             系统管理员准备好。该文件应该要全局可读。详细信息见sshd(8)。
    
     /etc/ssh/rc
             等同于~/.ssh/rc文件，包含了用户使用ssh登录成功，但启用shell
             (或指定命令执行)之前执行的命令。详细信息见sshd(8)的man文档。
             (译者注：也就是说，登录成功后做的第一件事就是执行该文件中的
             命令)

退出状态码
     ssh将以远程命令执行结果为状态码退出，或者出现错误时以255状态码退出。
```