---
title: zsh
author: ygdxd
type: 原创
date: 2019-10-14 20:19:56
tags:
---


###zsh

从 macOS Catalina 版开始，您的 Mac 将使用 zsh 作为默认登录 Shell 和交互式 Shell。您还可以在较低版本的 macOS 中将 zsh 设置为默认 Shell。

默认情况下，您的 Mac 使用 zsh 或 bash 作为登录 Shell 和交互式 Shell 的命令行解释器：

从 macOS Catalina Beta 版开始，zsh (Z shell) 是所有新建用户帐户的默认 Shell。
bash 是 macOS Mojave 及更低版本中的默认 Shell。
zsh 与 Bourne Shell (sh) 高度兼容，并且与 bash 基本兼容，但存在一些差别。要进一步了解 zsh 及其全面的命令行完成系统，请在“终端”中输入 man zsh。

在mac中打卡控制台会出现
The default interactive shell is now zsh.
To update your account to use zsh, please run `chsh -s /bin/zsh`.
For more details, please visit https://support.apple.com/kb/HT208050.

打卡网址发现新添加了.zprofile 和 .zshrc
那已经有了bash 为什么需要zsh?

######Licensing

google了一下发现https://thenextweb.com/dd/2019/06/04/why-does-macos-catalina-use-zsh-instead-of-bash-licensing/ 里提到苹果好像是为了更换里面的licensing。原来用的是GUN的bash，协议GPLv3。

同时苹果可能会开始维护更新这个zsh了。
