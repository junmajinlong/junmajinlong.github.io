---
title: Electron程序在Linux上启动不了的问题
p: others/electron_failed.md
date: 2023-07-10 18:20:41
tags: Others
categories: Others
---

# Electron程序在Linux上启动不了的问题解决

尝试加上以下几个选项：可一个一个试、结合起来试

- --no-sandbox
- --in-process-gpu
- --disable-gpu-sandbox

例如：
```
/opt/Bianance/binance --no-sanbox
proxychains /opt/Binance/binance --in-process-gpu --no-sandbox
```
