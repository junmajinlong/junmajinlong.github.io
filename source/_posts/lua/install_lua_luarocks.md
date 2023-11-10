---
title: 安装lua和luarocks
p: lua/install_lua_luarocks.md
date: 2020-05-02 10:40:35
tags: Lua
categories: Lua
---

--------

**回到：**  
- **[Lua系列文章](/lua/index)**  

--------

## 安装lua 5.3.4

```
yum install libtermcap-devel ncurses-devel libevent-devel readline-devel -y

curl -R -O http://www.lua.org/ftp/lua-5.3.4.tar.gz
tar xf lua-5.3.4.tar.gz
cd lua-5.3.4
make linux
make install INSTALL_TOP=/usr/local/lua53

# 如果已经装了lua51，先备份
mv /usr/bin/{lua,lua51}
mv /usr/bin/{luac,luac51}

ln -sf /usr/local/lua53/bin/lua /usr/bin/lua
ln -sf /usr/local/lua53/bin/luac /usr/bin/luac
```

## 安装luajit

```
wget https://github.com/LuaJIT/LuaJIT/archive/v2.0.5.tar.gz
tar xf v2.0.5.tar.gz
cd LuaJIT-2.0.5
make PREFIX=/usr/local/luajit2
make install PREFIX=/usr/local/luajit2

ln -sf /usr/local/luajit2/bin/luajit /usr/bin/luajit
```

## 安装luarocks

```
wget https://luarocks.org/releases/luarocks-3.3.1.tar.gz
tar xf luarocks-3.3.1.tar.gz
cd luarocks-3.3.1
./configure --prefix=/usr/local/luarocks
make && make install

ln -sf /usr/local/{luarocks,usr}/bin/luarocks
ln -sf /usr/local/{luarocks,usr}/bin/luarocks-admin
```







