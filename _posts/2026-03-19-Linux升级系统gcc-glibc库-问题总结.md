---
layout: mypost
title: Linux 升级gcc、glibc库问题总结
categories: [gcc]
---

### 目的
因为自己编译过Apache Gravitino和swanlake项目，这些项目在编译过程中都会因为系统自带的gcc版本过低导致无法运行。所以单开一篇文章记录下一些问题就当总结了，以后也方便直接在自己查找，省的到处查询其他网络文章了。

### 升级gcc
下面以centos7.9系统为例
在CentOS 7停止维护后，许多用户发现无法使用官方yum源进行软件包安装，特别是在尝试安装devtoolset-9-gcc和devtoolset-9-gcc-c++时遇到了域名解析问题。本文将详细介绍如何更换为国内镜像源，并成功安装所需的软件包。

### 解决方案
### 更换为国内镜像源
由于CentOS 7已停止维护，官方镜像源不可用。我们可以将yum源更改为国内第三方提供的镜像源，如阿里云、腾讯云、网易等。以下是具体步骤：

### 备份原始配置文件
```bash 
sudo cp /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.bak
sudo cp /etc/yum.repos.d/CentOS-SCLo-scl.repo /etc/yum.repos.d/CentOS-SCLo-scl.repo.bak
sudo cp /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo.bak
```

### 下载新的镜像源配置文件（以阿里云为例）
```bash
sudo wget -O /etc/yum.repos.d/CentOS-Base.repo https://mirrors.aliyun.com/repo/Centos-7.repo
```

### 编辑或创建阿里云的SCL配置文件
```bash
sudo vi /etc/yum.repos.d/CentOS-SCLo-scl-rh.repo
```
将内容替换为：

```bash
[centos-sclo-rh]
name=CentOS-7 - SCLo rh - mirrors.aliyun.com
baseurl=https://mirrors.aliyun.com/centos/7/sclo/x86_64/rh/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
```
```bash 
sudo vi /etc/yum.repos.d/CentOS-SCLo-scl.repo
```
将内容替换为：

```bash 
[centos-sclo-sclo]
name=CentOS-7 - SCLo sclo - mirrors.aliyun.com
baseurl=https://mirrors.aliyun.com/centos/7/sclo/x86_64/sclo/
gpgcheck=1
enabled=1
gpgkey=file:///etc/pki/rpm-gpg/RPM-GPG-KEY-CentOS-SIG-SCLo
```
### 清理Yum缓存并重建元数据
```bash 
sudo yum clean all
sudo yum makecache
```
### 验证仓库是否启用
```bash
yum repolist enabled | grep sclo
```
应该看到类似如下输出：

```bash 
centos-sclo-rh                      CentOS-7 - SCLo rh - mirrors.aliyun.com
centos-sclo-sclo                    CentOS-7 - SCLo sclo - mirrors.aliyun.com
```

### 安装devtoolset-9-gcc和devtoolset-9-gcc-c++
```bash 
sudo yum install -y devtoolset-9-gcc devtoolset-9-gcc-c++
```

### 启用devtoolset-9 临时
```bash 
scl enable devtoolset-9 bash
```

### 设置永久升级（可选）
```bash 
echo "source /opt/rh/devtoolset-9/enable" >> /etc/profile
```

### 升级系统glibc库
一般项目里面如果涉及到nodejs可以会因为glibc版本过低导致报错无法安装，还有就是有的项目里面依赖的库所需要的较高的glibc库(如：swanlake的duckdb)

首先查看自己服务器本地的库有没有较高的版本
命令如下：
```bash 
strings /lib64/libc.so.6 |grep GLIBC_
```
### 解决方案
两种方式任选一种

```bash 
sudo wget http://www.vuln.cn/wp-content/uploads/2019/08/libstdc.so_.6.0.26.zip
sudo curl -OL http://www.vuln.cn/wp-content/uploads/2019/08/libstdc.so_.6.0.26.zip
```
解压
```bash 
sudo unzip libstdc.so_.6.0.26.zip
```
安装
```bash 
cd /usr/lib64
ls -l | grep libstdc++
sudo rm libstdc++.so.6
sudo ln -s libstdc++.so.6.0.26 libstdc++.so.6
ls -l | grep libstdc++
```

后续还会继续补充

### 注意事项
以上需要服务器可以联网或者自己在电脑上下载好安装包再上传到服务器
