---
layout: post
categories: bug 
author: "Moevis"
published: true
---
### 描述场景

接到一个任务，要在 docker 中起网页后台和 mysql 服务，并配置 ssh 登陆，总之是做一个 all in one 的镜像（不要问我为什么要这样）。

在制作完成后，使用以下命令把容器保存为一个镜像。

```
$ docker commit xxx image-name:0.1
```

接着继续以该镜像启动一个容器：

```
$ docker run -it -p 10022:22 --name container-name image-name:0.1
```

但是在第二次启动时候出现问题，mysql 无法启动，我用的命令是这样的

```
$ service mysql start
 * Starting MySQL database server mysqld
   No directory, logging in with HOME=/   [fail]
```

## 解决

首先在网上搜，可能是 mysql 用户的主目录不对，所以改成这样：

```
$ usermod -d /var/lib/mysql/ mysql
```

但是这样依然运行不了

```
* Starting MySQL database server mysqld                              [fail]
```

后来我便想查看日志：

```
cat /var/log/mysql/error.log
```

发现崩溃的原因是这个：

```
Fatal error: Can't open and lock privilege tables: Table storage engine for 'user' doesn't have this option
```

于是我按照网上的方法，执行命令：

```
$ chown -R mysql:mysql /var/lib/mysql /var/run/mysqld
```

这样以后 mysql 就可以了正常跑了，虽然还没找到原因是啥。