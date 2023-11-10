## 部署hexo博客

```
sudo apt install -y npm 

npm install hexo-cli -g

cd /blog && npm install

cat /usr/lib/systemd/system/hexo.service
[Unit]
Description=hexo blog

[Service]
#MemoryLimit=250M
MemoryHigh=250M
CPUQuota=70%
WorkingDirectory=/blog
ExecStart=/usr/local/bin/hexo server -i 127.0.0.1
#Restart=always
#RestartSec=15s

[Install]
WantedBy=multi-user.target
```

## 安装mdbook

```
curl https://sh.rustup.rs -sSf | sh

source $HOME/.cargo/env
cargo install mdbook

cat /usr/lib/systemd/system/mdbook_rust.service
[Unit]
Description=rust book
AssertPathIsDirectory=/blog/books/rust

[Service]
#MemoryLimit=250M
MemoryHigh=40M
CPUQuota=70%
WorkingDirectory=/blog/books/rust
ExecStartPre=bash -c '/root/.cargo/bin/mdbook clean'
ExecStart=/root/.cargo/bin/mdbook serve --open -n 0.0.0.0 -p 8880
ExecStartPost=bash -c 'sleep 1;/usr/bin/cp mytheme/fonts/Cascadia.woff book/mytheme/fonts'
#Restart=always
#RestartSec=15s

[Install]
WantedBy=multi-user.target
```








1.安装python python-pip

```bash
apt install -y python python-pip ansible net-tools psmisc
```

2.下载blog和其中的ansible role

```bash
git clone 'https://github.com/malongshuai/blog.git' /blog
```

3.查看roles/blog_role/vars/main.yml中查看变量的设置，确定后执行

```bash
cd /blog/ansible_role
( ansible-playbook main.yml & )
```

4.cloudflare.com上设置dns到vps_ip  



配置swap

```
free -m 
dd if=/dev/zero of=/swap bs=1M count=1024
mkswap /swap
swapon /swap
echo "/swap  swap swap defaults 0 0" >> /etc/fstab
```







```
# 在新机器上
passwd root
apt install lrzsz zip unzip privoxy npm ufw net-tools socat
npm install hexo-cli -g

wget https://github.com/trojan-gfw/trojan/releases/download/v1.16.0/trojan-1.16.0-linux-amd64.tar.xz
tar xf trojan-1.16.0-linux-amd64.tar.xz
cp trojan/trojan /usr/bin

( curl https://sh.rustup.rs -sSf | sh ) && source $HOME/.cargo/env
cargo install mdbook

# 去安装证书

# 在旧机器上
tar zcvf util.tar.gz $HOME/utils
tar zcvf trojan.tar.gz /etc/trojan /usr/lib/systemd/system/trojan@.service 
tar zcvf privoxy.tar.gz /etc/privoxy/config  
tar zcvf ufw.tar.gz /etc/ufw       
tar zcvf nginx.tar.gz /etc/nginx /usr/share/nginx
tar --exclude node_modules -zcvf blog.tar.gz /blog /usr/lib/systemd/system/hexo.service /usr/lib/systemd/system/mdbook_*
scp *.tar.gz root@140.82.9.249:/root

# 在新机器上
tar -C / -xf nginx.tar.gz  
tar -C / -xf blog.tar.gz
tar -C / -xf trojan.tar.gz
tar -C / -xf privoxy.tar.gz
tar -C / -xf ufw.tar.gz
tar -C / -xf util.tar.gz
( cd /blog && npm install )
systemctl enable privoxy trojan@server trojan@client hexo mdbook_perl mdbook_rust mdbook_systemd nginx
systemctl start privoxy trojan@server trojan@client hexo mdbook_perl mdbook_rust mdbook_systemd nginx

curl https://www.cloudflare.com/ips-v4 | while read line; do ufw allow from $line;done
curl https://www.cloudflare.com/ips-v6 | while read line; do ufw allow from $line;done
```

