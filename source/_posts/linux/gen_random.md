---
title: shell生成指定长度的随机数
p: linux/gen_random.md
date: 2019-07-06 17:37:29
tags: Linux
categories: Linux
---

## 生成指定长度是随机数

```shell
# 8位纯数字的随机数
tr -cd '0-9' </dev/urandom | head -c 8

# 16位包含字母、数字的随机数
tr -cd '[:alnum:]' </dev/urandom | head -c 16
```

使用/dev/urandom而不是/dev/random是因为后者比较慢。
