---
title: 连接数据库SSLHandshakeException问题
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: 连接数据SSLHandshakeException问题
---

问题描述：
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E9%97%AE%E9%A2%98%E5%A4%84%E7%90%86/%E8%BF%9E%E6%8E%A5%E6%95%B0%E6%8D%AE%E5%BA%93SSLHandshakeException%E9%97%AE%E9%A2%98/shakehand%E9%94%99%E8%AF%AF.png)
在测试服务器上，java程序启动的时候，日志里面出现javax.net.ssl.SSLHandshakeException，这个错误目前还没有发现是什么原因导致的，大概率是有人升级了mysql或者jdk的版本。
在查找多方资料发现，是jdk8和mysql支持的协议不一致导致的。
mysql版本：5.7.34
jdk版本：1.8.0_292

问题剖析：
1.jdk8从小版本JDK8u261开始支持TLS1.3，默认TLS1.2之前的协议在security里面是禁用的，所以从JDK8u261开始只有TLS1.2和1.3是默认启用的
[jdk jsse连接](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html)
```
The JSSE API supports the following security protocols:
TLS: version 1.0, 1.1, 1.2, and 1.3 (since JDK 8u261)
SSL (Secure Socket Layer): version 3.0
```
2.mysql对于ssl的支持，我们看下官网上的描述:
[mysql连接](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-using-ssl.html)
```
TLS versions: The allowable versions of TLS protocol can be restricted using the connection properties enabledTLSProtocols and, for X DevAPI connections and for release 8.0.19 and later, xdevapi.tls-versions (when xdevapi.tls-versions is not specified, it takes up the value of enabledTLSProtocols). If no such restrictions have been specified, Connector/J attempts to connect to the server with the following TLS versions:
TLSv1,TLSv1.1,TLSv1.2,TLSv1.3 for MySQL Community Servers 8.0, 5.7.28 and later, and 5.6.46 and later, and for all commercial versions of MySQL Servers.
TLSv1,TLSv1.1 for all other versions of MySQL Servers.

Notes
For Connector/J 8.0.26 and later: The TLSv1 and TLSv1.1 protocols have been deprecated. While connections to the server using those TLS versions can still be made with the same negotiation process as described above, for any connections established using those TLS versions, Connector/J writes to its logger the message "This connection is using TLSv1[.1] which is now deprecated and will be removed in a future release of Connector/J."

For Connector/J 8.0.18 and earlier when connecting to MySQL Community Server 5.6 and 5.7 using the JDBC API: Due to compatibility issues with MySQL Server compiled with yaSSL, Connector/J does not enable connections with TLSv1.2 and higher by default. When connecting to servers that restrict connections to use those higher TLS versions, enable them explicitly by setting the Connector/J connection property enabledTLSProtocols (e.g., set enabledTLSProtocols=TLSv1,TLSv1.1,TLSv1.2).
```
这里很清楚的说明了，如果没有指定协议的情况下，在8.0.26以及之后的版本，TLS1.0和TLS1.1已经被废弃，但是可以作为握手的协议，会在mysql的日志记录器中提示这些协议会在未来被删除。
对于8.0.18以及之前的版本，包括5.6，5.7来说，为了兼容性的原因，不会启用1.2以及更高版本的TLS协议，所以mysql5.7这里默认是TLS1.0,TLS1.1被使用。
而jdk这边默认使用TLS1.2,TLS1.3。所以协议完全对不上，就在握手的时候报了SSLHandshakeException。
一下有三种解决方式：
第一种暴力的把jdk默认禁用安全协议给去掉了（或者只去掉TLS1.0和TLS1.1也可以），不太推荐使用，JDK默认的协议你给它直接暴力注释，总归是会带来一些安全问题。
第二种方式也很蛮力直接把ssl给去掉了，双方直接不使用ssl，也不太推荐使用，理由同上
第三种方式直接指定使用某种双方都支持的协议，mysql说了为了兼容问题不主动使用TLS1.2协议，这里确认没有兼容性问题，直接指定协议即可。

方案一：
```
在服务器输入命令：
which is java
返回：
/usr/bin/java

输入：ls -l /usr/bin/java
返回：/usr/bin/java -> /etc/alternatives/java

输入：ls -l /etc/alternatives/java
返回：/etc/alternatives/java -> /usr/lib/jvm/java-8-openjdk-amd64/jre/bin/java

然后知道java的目录之后，一步步进入到/usr/lib/jvm/java-8-openjdk-amd64/jre/lib/security
cd /usr/lib/jvm/java-8-openjdk-amd64/jre/lib/security

编辑java.sercurity
sudo vim java.security

然后搜索 SSLv3，找到 jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA,  类似的行
然后把这几行给注释掉
# Example:
#   jdk.tls.disabledAlgorithms=MD5, SSLv3, DSA, RSA keySize < 2048
# jdk.tls.disabledAlgorithms=SSLv3, TLSv1, TLSv1.1, RC4, DES, MD5withRSA, \
#    DH keySize < 1024, EC keySize < 224, 3DES_EDE_CBC, anon, NULL, \
#    include jdk.disabled.namedCurves

保存退出，然后重启java服务
```

方案二：
在连接数据库的地方设置不使用ssl，参数加上useSSL=false
```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/database?useSSL=false
```


方案三：
在连接后指定协议TLSv1.2
```
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/database?enabledTLSProtocols=TLSv1.2
```

参考资料：
1.[stackoverflow问题](https://stackoverflow.com/questions/67332909/why-can-java-not-connect-to-mysql-5-7-after-the-latest-jdk-update-and-how-should）
2.[mysql连接ssl](https://dev.mysql.com/doc/connector-j/8.0/en/connector-j-reference-using-ssl.html)
3.[JSSE](https://docs.oracle.com/javase/8/docs/technotes/guides/security/jsse/JSSERefGuide.html)
