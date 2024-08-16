---
title: hive的安装和配置
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: hadoop
---
#### hive的安装和配置
### 安装

1.先从官网下载hive的压缩包 [下载地址](http://www.apache.org/dyn/closer.cgi/hive/)
这里我下载的是hive-3.1.2
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/hive%E5%AE%89%E8%A3%85%E5%92%8C%E9%85%8D%E7%BD%AE/hive3.1.2%E5%AE%98%E7%BD%91.png)

2.配置hive，进入到hive的conf目录下，将默认的配置文件复制一份
```shell
cd hive/conf/
cp hive-default.xml.template hive-site.xml
vim hive-site.xml
```
配置如下：
```xml
<?xml version="1.0" encoding="UTF-8" standalone="no"?>
<?xml-stylesheet type="text/xsl" href="configuration.xsl"?>
<configuration>
    <property>
        <name>javax.jdo.option.ConnectionURL</name>
        <!-- mysql数据库中的地址  这里要先创建对应的数据库 -->
        <value>jdbc:mysql://mysql的host地址:3306/hive_test</value>
    </property>
        <!-- 数据库的账号的密码 -->
    <property>
        <name>javax.jdo.option.ConnectionDriverName</name>
        <value>com.mysql.jdbc.Driver</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionUserName</name>
        <value>root</value>
    </property>
    <property>
        <name>javax.jdo.option.ConnectionPassword</name>
        <value>XXXX</value>
    </property>
    <property>
        <name>hive.metastore.schema.verification</name>
        <value>false</value>
    </property>
</configuration>
```
这里要注意的就是hive_test是预先创建好的，文章中没有步骤，需要自己去创建一下对应的数据库

4.将mysql的驱动放入到hive目录的lib下，如果没有这个包可以从maven本地仓库去找或者maven的远程仓库，[maven远程仓库下载mysql驱动地址](https://mvnrepository.com/artifact/mysql/mysql-connector-java)


5.在hive中初始化mysql的数据库
```
schematool -dbType mysql -initSchema
```
这一步可能会报错，有可能是mysql版本问题或者是权限设置问题
比如说权限问题可能会报下面这个错：
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/hive%E5%AE%89%E8%A3%85%E5%92%8C%E9%85%8D%E7%BD%AE/%E6%9D%83%E9%99%90%E9%97%AE%E9%A2%98.jpg)

这里hive执行sql脚本会使用到外键，如果当前账号没有开启refernce的权限就回有问题。

6.启动hadoop，（这里默认大家之前都安装过hadoop了，如果没有的话可以参考[hadoop安装记录](https://juejin.im/post/5d3db3b2e51d457761476235)）
运行start-all.sh,启动完成后。

7.输入命令hive，然后
```
create datebase test ;
```
如果去mysql中hive_test查询
```
select * from DBS
```
DB_LOCATION_URI有对应的数据出现就是安装hive成功了~
