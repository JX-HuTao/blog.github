---
title: Virtualenv
categories: [python]
tags: [环境配置]
date: 2019-11-21 22:33:54
---
# 简介
virtualenv 用于创建独立的Python 开发环境的工具。从 Python 3.3 开始，部分功能已经集成到标准库下的 venv 模块下。

该工具主要用于解决不同程序依赖于相同模块的不同版本时的冲突问题。
# 安装
```sh
pip install virtualenv
```
# 使用
基本命令，其中 ENV 是用于存放虚拟环境的文件夹目录
```sh
virtualenv ENV
```

启用虚拟环境
```text
Windows 
    运行 ENV/Scripts/ 目录下的 activate.bat
Linux
    source /path/to/ENV/bin/activate
```

停用虚拟环境
```text
deactivate
```