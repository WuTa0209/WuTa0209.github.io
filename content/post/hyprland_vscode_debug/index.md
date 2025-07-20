---
title: "Hyprland VSCode 無法輸入中文解決"
description: 
date: 2025-07-20T12:27:11+08:00
image: 
math: 
license: 
categories:
    - Hyprland
hidden: false
comments: false
draft: false
---
## 修改 /etc/environment 以及 /etc/profile
以下為 `/etc/environment` 內容
```
XMODIFIERS=@im=fcitx
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
```
以下為 `/etc/profile` 內容
```
export XMODIFIERS=@im=fcitx
export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
```

接著只要執行 `code --enable-wayland-ime` 就可以正常輸入中文了

## 使用 alias
我這邊使用的 shell 是 zsh，加上 alias 讓上面的參數永久有效
```shell
$ echo "alias code='code --enable-wayland-ime'" >> ~/.zshrc
```