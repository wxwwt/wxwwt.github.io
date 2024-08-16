---
title: tomcat中post请求限制长度
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: basic
---
问题分析和解决:
有天测试的时候有一个接口,是同一份代码在预发环境上跑的很正常,但是在测试环境传入了两个参数(参数数据很长)后就返回了500错误.试了好几次发现传一个参数时正常,两个参数就报错,感觉很懵逼.(这里我还以为是代码问题,没有往环境问题上想),
抓包后发现一个参数的时候发送信息是1024之内,2个参数就超过了1024就报错了.我想难道是服务器限制了请求长度,然后去看了下测试服务器的配置,发下果然有人设置的tomcat接受的请求长度为1024.(真的蛋疼)
```xml
<Connector executor="tomcatThreadPool"  port="8080" protocol="HTTP/1.1"  connectionTimeout="20000"  redirectPort="8443" maxPostSize="1024"/> 
```
引用一下tomcat8的官方文档上说的:
The maximum size in bytes of the POST which will be handled by the container FORM URL parameter parsing.
The limit can be disabled by setting this attribute to a value less than zero. If not specified, this attribute is set to 2097152 (2 megabytes).
Note that the FailedRequestFilter can be used to reject requests that exceed this limit.
将 maxPostSize设置为-1(小于0,就是不限制post请求长度)就可以了.

总结:
1.其实这是一个很小的问题,生产上一般是不会设置这么小的,但是tomcat8默认是2M,如果真的请求参数超过了这个长度的还是会报错的.
ps:以前确实遇到过请求数据超过2M的post请求,也是修改tomcat的配置才可以;

2.这里历史还有一个小小的坑点就是,就是从tomcat7之后,不限制post请求长度都是小于0就可以了,等于0的话是长度不能超过0,也就是直接拒绝post请求.
tomcat6中是 maxPostSize小于等于0都是不限制,所以这个设置还是要先搞清楚自己用的tomcat是什么版本.(不过tomcat6应该没啥人用了吧,我也就简单的提一下罢了)
[tomcat6文档](http://tomcat.apache.org/tomcat-6.0-doc/config/http.html)
[tomcat7文档](http://tomcat.apache.org/tomcat-7.0-doc/config/http.html)
[tomcat8文档](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)
[tomcat8.5文档](http://tomcat.apache.org/tomcat-8.0-doc/config/http.html)
[tomcat9文档](http://tomcat.apache.org/tomcat-9.0-doc/config/http.html)
