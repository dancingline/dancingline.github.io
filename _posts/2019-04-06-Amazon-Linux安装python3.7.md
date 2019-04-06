---
layout: post
title: "Amazon Linux安装python3.7"
date: 2019-04-06
excerpt: 
tags: [Linux, Python]
feature: 
comments: false
---



最近薅了AWS免费一年的羊毛，起了个Amazon Linux AMI的实例，是个挺奇怪的Linux系统，上面自带的python是2.7的，想折腾成3.7，有一篇文章（https://blog.csdn.net/u012111475/article/details/80482697）说得比较全了，结合上面这个和一些新问题重新整理一个流程，另外参考的还有[[1]](https://blog.csdn.net/qq_36416904/article/details/79316972)和[[2]](https://blog.csdn.net/tanzuozhev/article/details/77585342)。

```bash
#安装依赖
sudo yum -y groupinstall development
sudo yum -y install zlib-devel
sudo yum -y install openssl-devel



#上面的openssl-devel装完openssl会报错，需要装一个相对新一点版本的openssl
wget https://github.com/openssl/openssl/archive/OpenSSL_1_0_2l.tar.gz
tar -zxvf OpenSSL_1_0_2l.tar.gz 
cd openssl-OpenSSL_1_0_2l/

./config shared
make
sudo make install
export LD_LIBRARY_PATH=/usr/local/ssl/lib/	#环境变量

#删除安装文件
cd ..
rm OpenSSL_1_0_2l.tar.gz
rm -rf openssl-OpenSSL_1_0_2l/



#安装python3.7.3
wget https://www.python.org/ftp/python/3.7.3/Python-3.7.3.tar.xz
tar xJf Python-3.7.3.tar.xz
cd Python-3.7.3

#3.7以上的版本需要一个新的libffi-devel，否则会报错错ModuleNotFoundError: No module named '_ctypes'
yum install libffi-devel -y

#安装
./configure
make
sudo make install

cd ..
rm Python-3.7.3.tar.xz
sudo rm -rf Python-3.7.3



#安装虚拟环境
sudo pip install --user --upgrade virtualenv	#--user是没权限的时候加的，否则应该不能在虚拟幻境里正常安装pip和setuptools
virtualenv -p python3 MYVENV
source MYVENV/bin/activate	#启动虚拟环境

#启动了虚拟环境后查看python版本号就是python3了
#正常在服务器里用python和pip是python2，而python3和pip3是python3的版本
```

