---
title: Matplotlib
categories: [python]
tags: [可视化]
date: 2019-11-20 20:48:13
---
# 配置镜像加速
1. 进入目录 %APPDATA%

2. 在该目录下新建 pip 文件夹，并在其中新建 pip.ini 文件

3. pip.ini 中写入
```text
[global]
index-url = https://mirrors.aliyun.com/pypi/simple
[install]
trusted-host=mirrors.aliyun.com
```

其他镜像：
```text
清华：https://pypi.tuna.tsinghua.edu.cn/simple/
中科大：https://pypi.mirrors.ustc.edu.cn/simple/
阿里云：https://mirrors.aliyun.com/pypi/simple/
豆瓣：http://pypi.doubanio.com/simple/
```
# 安装
```text
python -m pip install -U pip
python -m pip install -U matplotlib
```
# 示例
```python
import matplotlib.pyplot as plt
import numpy as np

x = np.arange(0, 10, 0.2)
y = np.sin(x)
fig, ax = plt.subplots()
ax.plot(x, y)
plt.show()
```
