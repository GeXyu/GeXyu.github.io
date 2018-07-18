---
title: 博客环境改为Liunx
comments: true
tags:
  - 博客环境改为Liunx
categories:
  - 博客环境改为Liunx
url: 226.html
id: 226
date: 2017-05-21 11:43:31
---

以前博客放在win环境下，phpstudy傻瓜式搞定（哈哈哈。但是最近出来莫名其妙的数据库访问不到或者是得到不到数据库的链接。google了一下，大部分的答案是网络防火墙问题要不就是重启一下，多次配置配置防火墙然后重启，治标不治本。索性把博客换成了Liunx，在搬博客的时候确实出了一些问题，把出现问题记录和安装配置环境的命令一下，方便下次搬（哈哈哈。

首先在win下把数据库导出和程序备份压缩阿，然后提交腾讯云工单重装成Linux系统（Ubuntu，主要是用着顺手）。然后其他乱七八糟的就略过，反正也是有些没有营养的东西。

**准备工作**
--------

### 1，重装完成后，登陆Ubuntu，想着先换个源吧。

vim /etc/apt/sources.list

![1](http://www.zzcode.cn/wp-content/uploads/2017/05/1.png)

看样子应该是腾讯云的源，我这股子机灵劲又出来了：我这个是腾讯云的主机，而腾讯云的源应该和我这个在同一个内网，所以还是不换了![devil](http://www.zzcode.cn/wp-content/plugins/ckeditor-for-wordpress/ckeditor/plugins/smiley/images/devil_smile.png "devil")

### 2，更新系统资源

sudo apt-get update && sudo apt-get upgrade

**LAMP环境**
----------

apache2

sudo apt-get install apache2

php7

sudo apt-get install php7.0

整合apahce和php7

sudo apt-get install libapache2-mod-php7.0
sudo service apache2 restart

现在可以测试一下apache和php环境是否可用，在**/var/www/html**文件夹下touch一个php文件。

touch 1.php
vim 1.php
<?php 
phpinfo();
?>

然后打开浏览器，输入外网地址，如果是本地那就是localhost。看到下面页面说明成功了。（当然，你也可以在**/etc/apache2/apache2.conf**文件中进行一些自定义的配置，譬如端口，网站程序位置等。我这里都是默认，别问我为什么，我不会告诉因为懒！）

![2](http://www.zzcode.cn/wp-content/uploads/2017/05/2.png)

继续安装mysql

sudo apt-get install mysql-server

当然mysql需要和php整合一下

sudo apt-get install php7.0-mysql

**其他**
------

环境配置OK了，接下来就是上传程序导入数据库了，我用的SecureCRT所以装一个**lrzsz**

sduo apt-get install lrzsz 

哎呀！一个rz命令，选中 文件 点击上传，美滋滋！

导入数据库，我用命令resource导入老是出错，如果用sqlyog远程连接，有需要改mysql的host。我怕出现一些玄学的问题。还是装一个**phpmyadmin**。

sudo apt-get install phpmyadmin 

安装完后默认的安装位置是在/usr/share 而不是在/var/www 所以 需要将其链接到/var/www/html来，复制的话貌似需要改配置文件，相当麻烦。所以我选择链接

sudo ln -s /usr/share/phpmyadmin /var/www/html/phpmyadmin

然后通过phpmyadmin导入数据库，全绿，看着就干爽。

**问题**
------

程序搬家成功了！紧接着问题出现了：发现主页可以访问，但是自定义的链接和文章并不能访问提示404。而这个404并不是wordpress的404页面而是服务器的404。

### **服务器404**

首先要用命令：

sudo a2enmod rewrite  
开启位于/etc/apache2/mods_avilable里面的rewrite模块

vim /etc/apache2/apache2.conf

  
​把<Directory /var/www/>里面的#AllowOverride None#改为#AllowOverride All# 

![20170521113754](http://www.zzcode.cn/wp-content/uploads/2017/05/20170521113754.png)

### **没有权限上传文件**

当对后台各个功能进行测试时，发现图片文件上传失败，提示没有权限。好吧，提高对uploads提高权限。

sudo chmod 777 uploads

OK！问题解决。(逃