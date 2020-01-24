---
title: "My Command Line Solution Roundup - 2019"
date: 2020-01-24T20:07:37+08:00
draft: true
tags: []
categories: []

toc: true
mathjax: false
---

在这个时间节点上，还是稍微来整理下过去这一年以来在命令行工具方面的积累吧。过去的一年也是我在命令行中工作地越来越多的一年——中学的时候写了不少东西，学了不少操作，但是现在看来，其实都非常地扁平；随着项目规模的扩大，还是需要更好地工具来武装自己，提高效率。

另外一点就是，有时候我们会觉得在图形环境下很容易工作，但在命令行环境下感到困难。所以我觉得一个比较有帮助的事情是，来整理一下我们如何使用一些图形的、集成的工具，从需求的角度出发，来整理命令行的使用技巧。

Here we go!

## Why Windows

往前两年特别痴迷于Linux环境，那时候觉得`nautilus`和`gnome-terminal`都是工作必备——我至今觉得`nautilus`是世界上最好的文件管理器，Linux上的Terminal选择范围也要大很多。但很遗憾，我的工作全部并不只是文件管理器和命令行，我还要上课，还要修图，还要摸鱼。所以，借着换新电脑的契机，我重新开始将Windows作为主要平台。

[Git for Windows](https://gitforwindows.org/)是个非常好的工具，提供了SSH和Git这种基本工具，还提供了个Bash工具箱；当然，没啥扩展性。微软近年来也推出了WSL，不过细想还是有很多坑，比如WSL的API不健全，桥接的文件系统，糟糕的IO性能，以及操作上其他GUI工具如何使用WSL中的工具？尤其是如果涉及到SSH和Git，其中其实是有一些坑点的；这些都并非不能解决的问题，但都是需要解决的问题，所以我还是在用着Git Bash，用着Native的Python。但WSL也并非没有用处，我依赖WSL来作为Git Bash的补充，在里面查看manpage、安装编译器等工具。两者互为补充，在大部分时候已经可以替代Linux了，不过我还是放了一个Xubuntu的虚拟机作为备用。

关于文件系统，NTFS上当然不存在权限设置，但是实际使用中这并不是一件特别糟糕的事情，Git for Windows默认设置了权限不敏感，如果要设置一个文件的权限可以用`update-index`，而且大部分时候其实都是`clone`现有的repo，所以对Git来说没啥影响；所以反而不要用WSL里的Git。软链的话，NTFS其实是支持的，可以在cmd里用`mklink`指令，而且Win10现在打开开发者模式就可以不需要管理员权限了。真正比较大的影响其实是路径大小写不敏感，放到NTFS上可能会出现路径相互覆盖。

总之，NTFS是个麻烦，但现在都能绕开了，应该说赶上了MSYS成熟、WSL启动的好时代；Linux上有的工具确实是神一般的好用，但代价是另外有的工具几乎就不可用。mac么，早年有人说他兼具Windows与Linux之长，但实际上也是兼具两者之短，键盘和接口上的遗憾也是很大的问题。

## Why Git Bash

关于Terminal的选择，我还是尝试了不少选择，最终最喜欢的还是Git for Windows自带的Git Bash (MinTTY)。简单，快速，功能齐全，除了没有标签页，没有什么好抱怨的。[MobaXterm](https://mobaxterm.mobatek.net/)也是个好东西，自带X Server，让我偶尔连集群不再需要Linux虚拟机或者x2go，但我对于启动速度并不怎么满意，其配置的眼花缭乱也令我敬而远之。对ConEmu的远离大抵也是因此，更何况我都没有把鼠标的问题解决出来。我也尝试过[hyper](https://hyper.is/)之类基于Electron的方案，在里面跑Git Bash的Shell，在解决了一些问题之后也还不错，但并不明显由于MinTTY吧。Windows Terminal初听觉得石破天惊，但是到现在仍然处于beta，也就不再提了。

值得一提的是，我很喜欢[Termius](https://www.termius.com/)，跨平台，可以同步（虽然我没有），而且还挺好看。不过在Windows客户端也没有标签页，虽然多窗口是有的，这本质上是个SSH客户端，但也可以执行本机shell，Git Bash是可以的，各种功能支持也还算健全。

前面说过，Git Bash是一个很不错的工具箱。SSH命令行，能够支持端口转发之类的功能，能够自己控制自己的密钥；sed、awk等脚本神器；一个简单的vim。我一度通过Git Bash来做一些Windows管理的工作，比如打包配置文件，还写了一些成规模的Shell脚本，但当然，我现在还是都回归了Native Python来完成这些工作——更好的可维护性，更好的性能，还是很香的。

更重要的一点是，Git Bash其实是个非常不错的Launcher。自带`winpty`，可以执行powershell和cmd，也可以安装[wslbridge](https://github.com/rprichard/wslbridge)，这个工具支持了广受赞誉的wsltty。

## 我的Git Bash配置

我基本上是香草（vanilla）派，外观上没有做太多的定制，从功能上倒是做了些修改。Git Bash主要还是用来连SSH，所以定制也还是从这方面着手。

开始入门SSH的时候基本上看的是GitHub的[手册](https://help.github.com/en/github/authenticating-to-github/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)，从生成密钥开始，后来又觉得明文密钥不安全，于是加上了口令。然而加入口令后麻烦其实真的不少：IDE里`git`不能用了，每次命令行连接也都要输密码，不过好歹每台服务器输的是同样的密码。

Git Bash没有标签页也算是个问题。我也试过基于Electron的方案，比如VSCode，Termius之类的，接着就发现中文也不支持了，`alias`也不全了。检查了一下，发现是login shell的问题。于是我开始在`.bashrc`里`source`一些基本的profile。

在这个过程中，发现了`start-ssh-agent.cmd`这么个脚本，自此开始折腾agent。想法很简单，开机启动agent，然后之后的`ssh`就都可以不用重复输密码了。但是Windows上不存在Login Shell，那环境变量该存在哪里呢？我试着用`setx`设Windows环境变量，效果很好，`git`也能够在VSCode里用了。但使用中还是有问题，经常会有智障清理软件把Socket File给删掉，这样的话agent就必须杀掉重新起；在VSCode里起起来的Git Bash会从Code继承旧的环境变量，结果还是会连不上agent。于是最近更新了一次，每次启动Git Bash都尝试启动agent，而且把环境存在一个文件中，这样每个子shell可以自己加载环境，不受继承的影响。

另外我还做了一个基于`.ssh/config`的SSH Host补全。上述的这些功能我都放在[我的GitHub](https://github.com/Tantalus13A98B5F/dotfiles/blob/master/bashrc.gitbash)上了。

## 我的Shell配置

## 我的tmux配置

## 我的编辑器选择
