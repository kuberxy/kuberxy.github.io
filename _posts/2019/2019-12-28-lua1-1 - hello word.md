---
layout: post
title: Lua入门(1) - Hello world
date:   2019-12-22 10:02:42 +0800
category: Lua
tags: [Lua]
typora-root-url: ../..
---



Lua，在葡萄牙语里代表美丽的月亮。事实证明它没有糟蹋这个优美的单词，Lua 语言正如它的名字所预示的那样成为了一门简洁、优雅且富有乐趣的语言。

## 简介

### Lua

Lua 从诞生之初就是作为一门可嵌入、可扩展的轻量级脚本语言来设计的，因此它一直遵从着“简单、小巧、可移植、快速”的原则，官方实现完全采用ANSI C 编写，它能以 C 程序库的形式嵌入到宿主程序中。LuaJIT 2 和标准 Lua 5.1解释器采用的是著名的 MIT 许可协议。

正是由于上述特点， Lua 在游戏开发、机器人控制、分布式应用、图像处理、生物信息学等各种各样的领域中得到了越来越广泛的应用。其中尤以游戏开发为最，许多著名的游戏，比如 Escape from Monkey Island、World of Warcraft、大话西游，都采用了 Lua 来配合引擎完成数据描述、配置管理和逻辑控制等任务。即使像 Redis 这样高性能的内存键值数据库也提供了内嵌用户 Lua 脚本的官方支持。



### Lua vs LuaJIT

Lua 非常高效，它运行得比许多其它脚本(如Perl、Python、Ruby)都快，这点在第三方的独立测评中得到了证实。尽管如此，仍然会有人不满足，他们总觉得“嗯，还不够快!”。LuaJIT 就是一个为了进一步榨出一些速度的尝试，它利用即时编译（Just-inTime）技术把 Lua 代码编译成本地机器码后交由 CPU 直接执行。

LuaJIT 2 的测评报告表明，在数值运算、循环与函数调用、协程切换、字符串操作等许多方面它的加速效果都很显著。凭借着 FFI 特性，LuaJIT 2 在那些需要频繁地调用外部 C/C++代码的场景，也要比标准 Lua 解释器快很多。目前 LuaJIT 2 已经支持包括 i386、x86_64、ARM、PowerPC 以及 MIPS 等多种不同的体系结构。

> LuaJIT = Lua解释器 + 即时编译器

LuaJIT 是采用 C 和汇编语言编写的 Lua 解释器与即时编译器。LuaJIT 被设计成全兼容标准的 Lua 5.1 语言，同时可选地支持 Lua 5.2 和 Lua 5.3 中的一些不破坏向后兼容性的有用特性。因此，标准 Lua 语言的代码可以不加修改地运行在 LuaJIT之上。

LuaJIT 和标准 Lua 解释器的一大区别是，LuaJIT 的执行速度，即使是其汇编编写的 Lua 解释器，也要比标准 Lua 5.1 解释器快很多，可以说是一个高效的Lua 实现。另一个区别是，LuaJIT 支持比标准 Lua 5.1 语言更多的基本原语和特性，因此功能上也要更加强大。



## 环境搭建

```shell
sudo apt install make gcc -y
wget http://luajit.org/download/LuaJIT-2.1.0-beta3.tar.gz
tar xf LuaJIT-2.1.0-beta3.tar.gz
cd LuaJIT-2.1.0-beta3
make && sudo make install
sudo ln -sf luajit-2.1.0-beta3 /usr/local/bin/luajit
luajit -v
```



## Hello World

```lua
-- print Hello world
print(1,'Hello world')
print(2,'Hello\\n world')
print(3,'Hello\n world')
print(4,'Hello\t world')
print(5,'Hello \
world')
```

注：在Lua中，注释用`--`标注



## 参考

https://moonbingbing.gitbooks.io/openresty-best-practices/content/













