---

layout:     post
title:      「环境配置」在Ubuntu系统上安装MySQL数据库
subtitle:   ubuntu+MySQL
date:       2022-03-22
author:     MRL Liu
header-img: img/the-first.png
catalog: True
tags: [环境配置]
   
---

[TOC]

​         本文讲解在Ubuntu系统上安装和配置MySQL数据库。该文参考自博客[Ubuntu安装MySQL_wavehaha的博客-CSDN博客_ubuntu下载mysql](https://blog.csdn.net/wavehaha/article/details/114730222)

## 一、安装MySQL-server

### 1、更新APT		

首先更新本地存储库索引，执行`sudo apt update`。

### 2、APT安装

​			然后从APT存储库安装MySQL，执行`suo apt install MySQL-server`，在安装过程中，可能会出现[Y / n]问题，输入Y继续。

​			如果上述安装出现问题，例如`E:无法定位软件包问题`。这时可以考虑使用命令`sudo apt-get install MySQL-server`

### 3、查看版本

​			安装完成后可以输入`mysql --version`来查看版本。

## 二、配置MySQL-server

为了提高MySQL安装的安全性，执行`sudo mysql_secure_installation`，会要求执行如下设置：

（1）设置密码。此步骤是为root用户设置安全密码，具体取决于选择的密码类型。 输入 Y并按Enter键 。 输入密码强度数字，然后按Enter键，输入密码即可。

（2）有关删除匿名测试用户的信息。按Y并按Enter键。

（3）关于禁止远程系统的root登录。建议root用户允许来自本地系统的连接，并拒绝来自远程连接的连接。 选择Y并按Enter键。
（4）删除“测试”数据库。 如果要删除它，请按Y并按Enter键.

（5）要求您重新加载特权表，以使上述更改生效。 按Y 键，然后按Enter键 ，所有安全设置将被提交.

全部完成后，将会显示all done。

## 三、登录MySQL-server

使用sudo命令连接到MySQL实例：`sudo mysql -u root -p`

然后输入密码即可登录。

