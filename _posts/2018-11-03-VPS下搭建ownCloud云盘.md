---
layout: post
title: VPS下搭建ownCloud云盘
date: 2018-11-03 18:09:01
tags:
    - ownCloud
    - vps
---

### 前言

鉴于各大网络云盘的各种限制，收费制度和用户体验并不好，利用VPS来搭建一个私有云盘就会变得越来越受欢迎，目前主流的开源软件ownCloud，NexCloud等都可以实现，下面我们就选ownCloud来搭建，vps选择的搬瓦工。

### 准备

1. 一台VPS服务器，开启root权限
2. LAMP环境。ownCloud需要Web服务器，数据库和PHP才能正常工作。 设置LAMP（Linux，Apache，MySQL和PHP）服务器满足所有这些要求
3. 安装ownCloud ，并配置相关参数。

### 设置LAMP环境

“LAMP”是一组开放源代码软件，通常安装在一起以使服务器能够托管动态网站和网络应用。这个词其实是代表linux下的操作系统，Apache Web服务器的缩写。 站点数据存储在MySQL数据库（使用MariaDB），以及动态内容用PHP处理。 接下来，我们将在CentOS 7 VPS上安装一个LAMP。 CentOS将满足我们的第一个需求：一个Linux操作系统。

- **安装Apache**

  在终端输入命令安装：

  ```shell
  sudo yum install httpd
  ```

  安装完成后，启动该服务

  ```shell
  sudo systemctl start httpd.service
  ```

  **踩坑**：如果无法启动报错，可能是没有设置防火墙对http/https的访问权限，输入下面命令开启

  ```shell
  sudo firewall-cmd --permanent --zone=public --add-service=http 
  sudo firewall-cmd --permanent --zone=public --add-service=https
  sudo firewall-cmd --reload
  
  // 注意：如果提示 firewall 没有启动，则使用命令开启：
  // 注意：如果开启防火墙后，ssh 无法访问远程vps，可以在vps服务商提供的web页面中的root shell 控制台来设置上述命令。
  
  启动： systemctl start firewalld
  关闭： systemctl stop firewalld
  查看状态： systemctl status firewalld 
  开机禁用  ： systemctl disable firewalld
  开机启用  ： systemctl enable firewalld
  ```

  再次使用命令来尝试开启httpd 服务。如果开启后(即没有报错信息)，下面设置开机默认启动：

  ```shell
  sudo systemctl enable httpd.service
  ```

  输入`http://你的服务器地址/`测试是否安装成功，如果能访问就说明安装没问题，如下截图：

  ![img0](https://user-gold-cdn.xitu.io/2017/12/13/1604ea6541926705?imageslim)

  到这里就Apache 服务就安装完成了。

- **安装MySQL（MariaDB）**

  现在我们已经开始运行Web服务器，现在是安装MariaDB的时候了，这是一个MySQL插件替换。 

  启动数据库以后会提示输入数据库密码。由于我们是首次安装，直接enter即可，同时会提示你设置密码，输入你想要设置的数据库密码即可。

  ```shell
  // 1. 安装数据库
  sudo yum install mariadb-server mariadb
  // 2. 启动数据库
  sudo systemctl enable mariadb.service
  // 3. 设置数据库，默认启动
  sudo systemctl enable mariadb.service
  ```

- **安装PHP**

  PHP是我们的设置的组件，它将处理代码以显示动态内容。它可以运行脚本，连接到我们的MySQL数据库以获取信息，并将处理的内容传递到我们的Web服务器以显示。 注意此处CentOS 7默认PHP为5.4版本，ownCloud需要的PHP版本为5.6以上。所以此处我们安装PHP5.6版本。

   执行下面命令升级php仓库

  ```shell
  rpm -Uvh https://mirror.webtatic.com/yum/el7/epel-release.rpm
  rpm -Uvh https://mirror.webtatic.com/yum/el7/webtatic-release.rpm
  ```

  执行安装

  ```shell
  yum install -y php56w php56w-opcache php56w-xml php56w-mcrypt php56w-gd php56w-devel php56w-mysql php56w-intl php56w-mbstring  php56w-openssl php56w-Tokenizer php56w-posix php56w-pcntl
  ```

  **这里要特别注意**，可能会报错php56w-commen 和php5.4-commen发生文件冲突，导致安装不成功。因为centOS 7 系统默认会安装php5.4，如果直接执行安装高版本的话就会报错，通过踩坑经验最简单直接的方法就是删除掉原有的php5.4，在安装php56w或更高的版本：

  删除旧版本php的方法命令：

  ```shell
  yum remove php-common
  ```

  再次执行上面的安装命令，如果安装成功，执行下面命令重启httpd服务

  ```shell
  sudo systemctl restart httpd.service
  ```

  还可以通过执行命令`php -v`查看php的安装版本

  为了测试PHP是否正确配置，可以创建一个php脚本来查看。在web根目录`/var/www/html/`创建一个info.php 文件，然后使用vim打开这个文件插入下面这句话保持退出：

  ```
  <?php phpinfo(); ?>
  ```

  然后访问 `http://你的服务器地址/info.php` 如果没有问题页面会展示PHP的一些基本信息。 最后不忘删除我们的测试页面

  ```shell
  sudo rm /var/www/html/info.php
  ```

  到这里整个LAMP服务就都安装完成了。

### 安装ownCloud

ownCloud服务器软件包不存在于CentOS的默认存储库中。然而，ownCloud为发行版维护了一个专用的存储库。 首先，导入与他们释放钥匙rpm命令。 关键的授权包管理器yum信任库。

```shell
sudo rpm --import https://download.owncloud.org/download/repositories/stable/CentOS_7/repodata/repomd.xml.key
```

接下来，使用curl命令下载ownCloud库文件：

```shell
sudo curl -L https://download.owncloud.org/download/repositories/stable/CentOS_7/ce:stable.repo -o /etc/yum.repos.d/ownCloud.repo
```

添加新文件后，用clean命令使yum知道所做的更改：

```shell
sudo yum clean expire-cache

// 如果提示类似下列命令，说明执行成功了
OutputLoaded plugins: fastestmirror
Cleaning repos: base ce_stable extras updates
6 metadata files removed
```

最后，使用进行ownCloud安装yum实用程序和install命令：

```powershell
sudo yum install owncloud
// 当提示Is this ok [y/d/N]:消息类型Y然后按ENTER键授权安装
```

**配置数据库**

```powershell
mysql -u root -p

// 可能会提示要输入密码，如果没有直接enter即可
```

为ownCloud 创建表，注意：每个MySQL的语句必须以分号`;`结束

```sql
CREATE DATABASE owncloud;
```

接下来，创建一个单独的MySQL用户帐户，与新创建的数据库进行交互。从管理和安全的角度来看，这样做不仅有利于数据安全更有利于我们日后的管理工作。与数据库的命名一样，选择您喜欢的用户名。我们选择owncloud

```sql
GRANT ALL ON owncloud.* to 'owncloud'@'localhost' IDENTIFIED BY '此处填写你想要设置的密码';
```

执行flush-privileges操作以确保MySQL应用权限分配

```shell
FLUSH PRIVILEGES;
```

数据库已经完成配置，执行命令`exit`退出。

### 配置ownCloud

打开`https://你的服务器地址/owncloud`进入web管理页面 此时页面会提示你创建管理员账号，输入你想要的管理员账号和密码。在下方的数据库选项中选择MySQL／MariaDB，并且填入相应的账号和密码。此处填入的账号和密码即之前我们设置数据库时设置的账号和密码

![img1](https://user-gold-cdn.xitu.io/2017/12/13/1604ea6541bffadc?imageslim)

点击Finish setup 完成安装。然后重新进入ownCloud web页面就可以进入到网盘。

![img2](https://user-gold-cdn.xitu.io/2017/12/13/1604ea6541bb082a?imageslim)

到这里整个安装流程就都完成，可以开心的上传自己的文件啦。