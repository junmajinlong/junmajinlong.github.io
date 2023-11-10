---
title: Shell脚本深入教程：Shell脚本带颜色输出
p: shell/script_course/shell_color.md
date: 2020-05-19 14:17:18
tags: Shell
categories: Shell
---

------

**[Linux基础系列文章大纲](/linux/index)**  
**[Shell系列文章大纲](/shell/index)**  

------

# Shell脚本带颜色输出

如果一个脚本功能较多较复杂，那么脚本的人性化输出将显得非常重要。

下面是一个各种带颜色的输出函数，本质都是echo，用法也和echo一样。
```shell
#====   Colorized variables  ====
if [[ -t 1 ]]; then # is terminal?
  BOLD="\e[1m";      DIM="\e[2m";
  RED="\e[0;31m";    RED_BOLD="\e[1;31m";
  YELLOW="\e[0;33m"; YELLOW_BOLD="\e[1;33m";
  GREEN="\e[0;32m";  GREEN_BOLD="\e[1;32m";
  BLUE="\e[0;34m";   BLUE_BOLD="\e[1;34m";
  GREY="\e[37m";     CYAN_BOLD="\e[1;36m";
  RESET="\e[0m";
fi

function title() { echo -e "${BLUE_BOLD}# ${1}${RESET}"; }
function finish() { echo -e "\n${GREEN_BOLD}# Finish!${RESET}\n"; exit 0; }
function userAbort() { echo -e "\n${YELLOW_BOLD}# Abort by user!${RESET}\n"; exit 0; }
function warn() { echo -e "${YELLOW_BOLD}  Warning: ${1} ${RESET}"; }
function success() { echo -e "${GREEN}  Success: ${1} ${RESET}"; }
function error() { echo -e "${RED_BOLD}  Error:   ${RED}$1${RESET}\n"; exit 1; }
```

将上面的内容保存在一个文件中，比如`color_print.sh`，所有这类工具类的脚本建议都统一存放，比如`~/shlibs`目录下，以后使用时直接source接口。例如：
```shell
#!/bin/bash

source ~/shlibs/color_print.sh

title "Downloading..."
title "Service start..."
success "download file to /tmp"
error "download file to /tmp"
rm "$TARGET_FILE" || warn "delete temp file failed: $TARGET_FILE"
finish
```

此外，还可以使用一些比较奇特的输出方式：
```shell
function throw() { echo -e "[-] FATAL: $1\nexit with code 1"; exit 1; }

echo '[.] copying `FILENAME` ...'
cp /etc/fstab "aaa/" || throw 'Copy `FILENAME` failed!'
echo '[~] copied `FILENAME`'
echo "[+] success: installed to aaa/"
```