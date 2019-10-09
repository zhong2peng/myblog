---
title: ssr服务器
date: 2019-04-10 21:30:46
tags:
---
# 搭建步骤
* 创建ssh秘钥登录
```
cd ~/ && mkdir .ssh && cd .ssh && yum install -y lrzsz && rz
```
* ssr安装
```
wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/ssr.sh && chmod +x ssr.sh && bash ssr.sh
```
* 安装加速器，会重启
```
wget --no-check-certificate https://blog.asuhu.com/sh/ruisu.sh && bash ruisu.sh
```
* 加速器
```
cd ~/.ssh
wget -N --no-check-certificate https://raw.githubusercontent.com/91yun/serverspeeder/master/serverspeeder-all.sh && bash serverspeeder-all.sh
```
* 查看配置信息
```
./ssr.sh 
```

cd ~/ && mkdir .ssh && cd .ssh && yum install -y lrzsz && rz
wget -N --no-check-certificate https://raw.githubusercontent.com/ToyoDAdoubi/doubi/master/ssr.sh && chmod +x ssr.sh && bash ssr.sh
wget --no-check-certificate https://blog.asuhu.com/sh/ruisu.sh && bash ruisu.sh
ZPzp520520
ZPzpZPzp520@