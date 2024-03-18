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






