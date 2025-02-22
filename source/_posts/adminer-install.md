---
title: Adminer轻量级MySQL管理工具
author: ciey
categories: 常用工具
tags:
  - Adminer
  - MySQL
date: 2018-12-24 13:35:00
---
![image](https://user-images.githubusercontent.com/3664948/63909091-103ce280-ca54-11e9-96e9-7a4f63fede37.png)

#### Adminer

工具名称：Adminer

工具作用：图形化MySQL管理工具

工具下载： [https://www.adminer.org/](https://www.adminer.org/)


**Why is Adminer better than phpMyAdmin?**

优势：1. 安全, 2. 用户体验更好, 3. 性能更好, 4. 支持特性更多, 5. 最小体积，单个文件.

附Adminer与phpMyAdmin的对比[https://www.adminer.org/en/phpmyadmin/](https://www.adminer.org/en/phpmyadmin/)



## Apache 安装


```
$ sudo apt-get install apache2
```

## php 安装

```
$ sudo apt-get install php7.0
$ sudo apt-get install libapache2-mod-php7.0
```

## mysql 另行安装，不在赘述

```
$ sudo apt-get install mysql-server mysql-client
```

## php扩展安装

```
$ sudo apt-get install php7.0-mysql
```

## adminer安装

下载adminer.php至/var/www/html
```
$ cd /var/www/html
$ sudo wget "http://www.adminer.org/latest.php" -O adminer.php
```

重启Apache
```
$ sudo /etc/init.d/apache2 restart
```

重载Apache配置
```
$ sudo systemctl reload apache2
```

![image](https://user-images.githubusercontent.com/3664948/63909031-eb486f80-ca53-11e9-8e5b-19a76e5ee604.png)