---
title: pip更换为国内的源
date: 2020-03-30 13:59:03
tags:
	- python
	- pip
excerpt: pip更换为国内的源
---
# pip修改源
### pip国内的一些镜像
- 阿里云 https://mirrors.aliyun.com/pypi/simple/
- 中国科技大学 https://pypi.mirrors.ustc.edu.cn/simple/
- 豆瓣(douban) http://pypi.douban.com/simple/
- 清华大学 https://pypi.tuna.tsinghua.edu.cn/simple/
- 中国科学技术大学 http://pypi.mirrors.ustc.edu.cn/simple/

---

### 修改源方法
1. 临时使用
```shell
# 可以在使用pip的时候在后面加上-i参数，指定pip源
pip install scrapy -i https://pypi.tuna.tsinghua.edu.cn/simple
```

2. 永久修改
	Linux:
	```shell
	# 修改 ~/.pip/pip.conf (没有就创建一个)
	[global]
	index-url = https://pypi.tuna.tsinghua.edu.cn/simple
	```

	Windows:

	```shell
	# 直接在user目录中创建一个pip目录，如：C:\Users\xx\pip，新建文件pip.ini，内容如下
	[global]
	index-url = https://mirrors.aliyun.com/pypi/simple/
	[install]
	trusted-host=mirrors.aliyun.com
	```

---


# conda修改源
conda是anaconda的包管理器，通过conda可以从软件源中下载用户制定的软件及其依赖软件并在用户的系统上进行安装。

这里要说的是，conda的官方源因为服务器在国外，所以速度是很慢的。这里介绍给conda换成国内软件源的方式。这里使用的是清华大学计算机协会（tuna）提供的软件源。

也有一些其他机构提供了conda的软件源镜像，但是我没搜到相关的官方文档，所以这里只介绍tuna的，有其他需求的用户可以自行查找。

直接安装conda之后，执行下述命令就可以
```shell
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/
conda config --set show_channel_urls yes
```
### miniconda
miniconda是anaconda的一个精简版（或者叫轻量级替代）。anaconda安装包里面带了大部分科学计算用的软件包还有ide，所以安装包很大，但是这些玩意儿并不是所有用户都用的到。而且很多用户装完anaconda之后，还是要联网安装一些自己需要的，anaconda安装包里面没有的软件。所以就不如直接安装miniconda，里面有python和conda（全功能的conda，跟anaconda里面的是一样的，这个不是精简的），然后根据自己需要定制安装软件。

### 参考
Anaconda 镜像使用帮助: https://mirror.tuna.tsinghua.edu.cn/help/anaconda/

---