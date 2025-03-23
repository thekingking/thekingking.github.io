---
title: "Docker配置一条龙"
date: 2025-03-22T10:18:12+08:00
lastMod: 2025-03-23T10:18:12+08:00
draft: false # 是否为草稿
author: ["tkk"]

categories: [docker, 配置]

tags: [docker, 配置]

keywords: [docker, 配置]

description: "docker环境配置记录" # 文章描述，与搜索优化相关
summary: "docker环境配置记录" # 文章简单描述，会展示在主页
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

## 创建docker

```bash
# 查看已有docker容器，不要和其他人的容器重复了（如名称、端口）
docker ps -a

# 查看已有docker镜像，选择你需要的镜像创建容器
docker images

# 创建docker容器，docker_name 指定容器名，port指定ssh映射端口，images_name 指定镜像
docker run -itd --name docker_name --gpus all -p port:22 images_name

# 查看是否创建成功，显示正在运行的容器
docker ps
```

解释下上述docker创建命令内容：

- docker run
  - 基于指定镜像创建并启动一个新容器
- -itd:
  - ​-i：保持标准输入（STDIN）开启，允许与容器交互（如输入命令）
  - -t：分配伪终端（pseudo-TTY），使容器支持终端操作（如命令行界面）
  - ​-d：以“守护进程模式”（后台运行）启动容器
- -v /home/xxx/.ssh:/root/.ssh
  - 将宿主机的目录 /home/xxx/.ssh 挂载到容器内的 /root/.ssh 目录
- --name yyy_ubuntu
  - 为容器指定唯一名称 yyy_ubuntu，替代默认生成的随机名称
- --gpus all
  - 允许容器使用宿主机上的所有 GPU 资源
- -p port:22
  - 指定ssh映射端口
- ubuntu:22.04
  - 指定镜像来源

> **目前发现把主机.ssh目录挂载到docker后配置ssh连接存在问题，尚未搞清楚原因**

## 删除docker

```bash
# 停止容器
docker stop docker_name

# 查看所有容器，查看是关闭
docker ps -a

# 删除容器
docker rm docker_name
```

## ssh配置

```bash
# 进入容器
docker exec -it docker_name /bin/bash

# 安装ssh服务
apt update && apt install -y openssh-server

# 安装vim，有的话就不用安装了
apt-get install vim -y
```

vim使用技巧：可以使用`/`进行搜索，按下`Enter`后进入搜索，按`n`或`N`向下或向上搜索

```bash
# 编辑ssh配置文件
vim /etc/ssh/sshd_config
```

将文件中如下内容解除注释(按`x`删除单个字符)：

```ssh
# Personal Config
Port 22
PermitRootLogin prohibit-password
PubkeyAuthentication yes      # 开启秘钥登录
AuthorizedKeysFile .ssh/authorized_keys
```

```bash
# 创建.ssh目录并修改权限
cd ~
mkdir .ssh
chmod 700 .ssh

# 将本地公钥复制到authorized_keys文件中
vim .ssh/authorized_keys

# 启动ssh服务
service ssh start

# 查看ssh服务运行状态
service ssh status
```

## git配置

```bash
# 安装git
sudo apt install git

# git配置
# 验证安装
git --version
# git配置 
git config --global user.name "Your Name" 
git config --global user.email "youremail@domain.com"
# 查看git配置
git config --list
# 清除配置
git config --global unset "错误属性"
# 生成秘钥，将公钥传到github上
ssh-keygen -t rsa
# 测试ssh连接
ssh -T git@github.com
```

## pytorch容器配置

[conda配置](https://thekingking.github.io/posts/conda%E5%9F%BA%E7%A1%80%E6%93%8D%E4%BD%9C/)

```bash
# 安装gcc编译器
sudo apt-get install build-essential python3-dev libicu-dev
# 更新pip和setuptools确保兼容性
python -m pip install --upgrade pip setuptools wheel
```
