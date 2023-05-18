+++
title = "生成 Qt 项目的编译数据库（compile_commands.json）"
date = 2023-01-25T23:11:39+08:00
author = "NKID00"
tags = ["GNU/Linux", "Qt"]
hideComments = false
+++

需要安装 Qt, make 和 bear，先使用 qmake 生成 makefile。

```sh
qmake project.pro
```

再使用 bear 根据 make 的输出生成编译数据库。

```sh
bear -- make
```
