---
title: "Conda基础操作"
date: 2025-03-23T10:02:43+08:00
lastMod: 2025-03-23T10:02:43+08:00
draft: false # 是否为草稿
author: ["tkk"]

categories: [conda, python]

tags: [conda, python]

keywords: [conda, python]

description: "conda 基础操作记录" # 文章描述，与搜索优化相关
summary: "conda 基础操作记录" # 文章简单描述，会展示在主页
weight: # 输入1可以顶置文章，用来给文章展示排序，不填就默认按时间排序
slug: ""
comments: false
autoNumbering: true # 目录自动编号
hideMeta: false # 是否隐藏文章的元信息，如发布日期、作者等
mermaid: true
cover:
    image: ""
    caption: ""
    alt: ""
    relative: false
---

## conda初始化

```bash
# 将 Conda 默认启动配置到终端设置中
conda init bash
source ~/.bashrc
```

如果配置成功，终端开头应该会像下面这样显示

```bash
(base) $
```

## 配置国内镜像源

[清华大学镜像源](https://mirrors.tuna.tsinghua.edu.cn/help/anaconda/)

```bash
vim ~/.condarc
```

配置文件内容如下（后续可能还会改，先记录下）：

```config
channels:
  - conda-forge  # 必须置于首位
  - defaults
show_channel_urls: true
custom_channels:
  conda-forge: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
  pytorch: https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud
```

```bash
# 更新conda配置
conda clean -i
conda update conda -y

# 查看是否配置成功
conda config --show-sources  # 应显示修改后的频道顺序
conda config --get channels  # 输出应为['conda-forge', 'defaults']
```

## conda 包管理

```bash
# 安装包
conda install package_name

# 查看已安装的包
conda list 

# 更新包
conda update package_name

# 删除包
conda remove package_name
```

## Python 版本管理

```bash
# 列出可安装的Python版本
conda search python

# 创建指定版本的python环境、安装包
conda create -n env_name python=3.9.10 numpy pandas

# 激活环境
conda activate env_name

# 退出当前环境
conda activate

# 列出所有环境
conda env list

# 删除指定环境
conda env remove -n env_name
```

## conda 项目迁移
