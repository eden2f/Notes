# Linux 安装Redis

## SQL与NoSQL 

SQL，即结构化查询语言，是传统的关系型数据库的查询语言。SQL数据库能够通过简化CRUD操作，处理数据库中的结构化数据。此处的CRUD代表了创建(create)、检索(或读取，retrieve、read)、更新(update)和删除(delete)，四种控制数据的主要操作。

SQL数据库通常被称为关系型数据库管理系统(RDBMS)。由于此类系统主要利用基于行的数据库结构，连接各个数据表之间的相关数据对象，因此传统的RDBMS使用的是SQL语法。我们熟悉的Microsoft Access、MySQL、Microsoft SQL Server、SQLite、Oracle Database、IBM DB2、以及Backendless等都是RDBMS类型的SQL数据库。

NoSQL，泛指非关系型的数据库。随着互联网web2.0网站的兴起，传统的关系数据库在处理web2.0网站，特别是超大规模和高并发的SNS类型的web2.0纯动态网站已经显得力不从心，出现了很多难以克服的问题，而非关系型的数据库则由于其本身的特点得到了非常迅速的发展。NoSQL数据库的产生就是为了解决大规模数据集合多重数据种类带来的挑战，特别是大数据应用难题。

## Redis简介

Redis（Remote Dictionary Server )，即远程字典服务，是一个开源的使用ANSI C语言编写、支持网络、可基于内存亦可持久化的日志型、Key-Value数据库，并提供多种语言的API。

## 示例环境

* OS CentOs 6.5
* Redis 4.0.1

## 在Linux上安装Redis

1. 安装redis编译的c环境，yum install gcc-c++
2. 将 redis-4.0.1.tar.gz 上传到Linux系统中
3. 解压到/usr/local下  tar -xvf redis-4.0.1.tar.gz -C /usr/local
4. 进入 /usr/local/redis-4.0.1 目录, 使用 make 命令编译 redis
5. 在 redis-4.0.1 目录中 使用 make PREFIX=/usr/local/redis install 命令安装			redis 到 /usr/local/redis 中
6. 拷贝 redis-4.0.1 中的 redis.conf 到安装目录 /usr/local/redis/bin 中
7. 启动 redis  在 bin 下执行命令

``` 
./redis-server redis.conf
```

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126233547.png)
        
注意 : 启动后看到如上欢迎页面，但此窗口不能关闭，窗口关闭就认为redis也关闭了(类	似Tomcat通过bin下的startup.bat的方式)
    
## 其他设置

### 让Redis允许被远程访问

* 修改 /usr/local/redis/bin/redis.conf 文件, 注释 bind 属性
``` 
sudo vi /usr/local/redis/bin/redis.conf

# bind 127.0.0.1
```
### 为Redis设置登录密码

* 修改 /usr/local/redis/bin/redis.conf 文件, 取消注释 redisredis 属性,并赋值

``` 
sudo vi /usr/local/redis/bin/redis.conf

redisredis 密码
```

### 用Redis Desktop Manager远程访问Redis

* 填写连接参数

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126233740.png)
        
* 连接成功

![](https://eden-notes-pic-hosting.oss-cn-shenzhen.aliyuncs.com/notes/images/20240126233816.png)