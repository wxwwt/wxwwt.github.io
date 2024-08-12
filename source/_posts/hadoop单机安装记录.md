### 引语:
&nbsp;&nbsp;&nbsp;&nbsp;最近想接触一些大数据相关的技术,所以有了这篇文章,其实就是记录一下自己学习hadoop的过程,如果文章中有啥写的不对的地方,还望指正(有java开发经验,但是是大数据小白一只,各位大神轻喷.)
我先是在网上搜索了一波大数据应该要学些什么技术,基本上不约而同的都是指向了hadoop.
&nbsp;&nbsp;&nbsp;&nbsp;摘自维基百科:[Apache Hadoop链接地址](https://zh.wikipedia.org/zh-hans/Apache_Hadoop).看完维基百科描述,我们大概知道了hadoop是一个分布式的大数据框架，在深入一些我们会知道
它是由很多个组件组成的（比如核心的HDFS，Hadoop Distributed File System，Mapreduce框架，还有很多Hive，HBase等等）。所以hadoop其实也是代指hadoop的一套的生态系统。光说不练假把式，好的我们来看看怎么安装，搭建hadoop的环境呢？

### 安装步骤：
&nbsp;&nbsp;&nbsp;&nbsp;这里其实有个前提，默认各位大佬的机器上已经安装好了linux和java环境。如果没有可以动动您灵活的手指，在搜索栏敲下“如何安装linux/java环境”,不开玩笑了，这个比较常见。
hadoop安装官网上说是有三种方式：
1.单机模式安装
2.伪分布式安装
3.全分布式安装（真.分布式）
我这里使用的是伪分布式，有人要问为啥不用真.分布式呢？
第一是初学者学会了伪分布式，基本上全分布式也是不会有大问题的，只是机器多了；
第二是因为贫穷，
![](https://user-gold-cdn.xitu.io/2019/7/28/16c3905e69d9cd25?w=240&h=210&f=jpeg&s=8476)
在云服务器上搭建的，全分布式要搞好几台。2333~ 开个玩笑啦,主要是懒~
#### 1.下载hadoop
去官网下载就可以了，[官网链接](http://hadoop.apache.org/)。
![](https://user-gold-cdn.xitu.io/2019/7/28/16c3905e69c5e216?w=1466&h=434&f=png&s=73361)
点击source，然后在跳转的页面中下载hadoop-3.1.2-src.tar.gz。
```
tar -zxvf hadoop-3.1.2-src.tar.gz
```
然后进入到 hadoop-3.1.2中，目录是这样
![](https://user-gold-cdn.xitu.io/2019/7/28/16c3905e6a8701a0?w=146&h=256&f=png&s=5541)
bin和sbin是可执行脚本的目录，etc是放hadoop配置文件的目录。

#### 2.配置hadoop
(1).首先先配置hadoop的环境文件hadoop-env.sh，进入到hadoop-3.1.2/etc/hadoop目录下，编辑 hadoop-env.sh文件
然后搜索JAVA_HOME,会发现两处，但是可以通过阅读英文注释得知是哪一个
```
# Technically, the only required environment variable is JAVA_HOME.
# All others are optional.  However, the defaults are probably not
# preferred.  Many sites configure these options outside of Hadoop,
# such as in /etc/profile.d

# The java implementation to use. By default, this environment
# variable is REQUIRED on ALL platforms except OS X!
```
在这下面添加export JAVA_HOME=“机器上的jdk地址”

在环境变量中添加hadoop的配置，vim /etc/profile,添加hadoop_home
```
export HADOOP_HOME=/home/hadoop/hadoop-3.1.2
export PATH=$PATH:$JAVA_HOME/bin:$HADOOP_HOME/sbin:$HADOOP_HOME/bin
```


(2).在配置hadoop的核心配置文件core-site.xml，这里配置的是hdfs的NameNode地址和数据存储目录路径
在<configuration>里面添加：
```
这个是hdfs的地址
<property>
       <name>fs.defaultFS</name>
       <value>hdfs://wxwwt-hadoop:9000</value>
   </property>
hadoop存储数据路径
<property>
     <name>hadoop.tmp.dir</name>
     <value>/home/hadoop/hadoop-3.1.2/data</value>
</property>
```
wxwwt-hadoop这个是自己随便取得名字，记得在etc/hosts中配置一下，映射到本地127.0.0.1

(3).配置mapred-site.xml，从名字上就能看出来是和MapReduce相关的。指定一下调度系统的配置
```
<property>
      <name>mapreduce.framework.name</name>
      <value>yarn</value>
</property>
```

(4).配置yarn-site.xml
```
<configuration>
指定ResouceManager的地址
    <property>
       <name>yarn.resourcemanager.hostname</name>
       <value>wxwwt-hadoop</value>
    </property>
指定MapReduce的方式
    <property>
       <name>yarn.nodemanager.aux-services</name>
       <value>mapreduce_shuffle</value>
    </property>
</configuration>
```
到这里已经上hadoop的配置文件是弄完了

#### 3.格式化文件系统
```
hdfs namenode -format
```
如果能看到类似下面的信息，说明格式化成功了
```
Storage directory /home/hadoop/hadoop-3.1.2/data/dfs/name has been successfully formatted.
```

#### 4.运行hadoop，start-dfs.sh
不过在运行之前，先得说一句如果之前一直使用的root用户的话，这里运行会报错。因为会因为是root用户，hadoop不建议使用root用户。会报出
```
ERROR: Attempting to operate on hdfs namenode as root
ERROR: but there is no HDFS_NAMENODE_USER defined. Aborting operation.
Starting datanodes
ERROR: Attempting to operate on hdfs datanode as root
ERROR: but there is no HDFS_DATANODE_USER defined. Aborting operation.
```
类似的错误。
(1).如果想用root用户继续执行的话，得在启动脚本start-dfs.sh中添加
```
HDFS_DATANODE_USER=root
HADOOP_SECURE_DN_USER=hdfs
HDFS_NAMENODE_USER=root
HDFS_SECONDARYNAMENODE_USER=root
```
并且要设置root的免密登录。
步骤：
```
$ ssh-keygen -t rsa -P '' -f ~/.ssh/id_rsa
$ cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys
$ chmod 0600 ~/.ssh/authorized_keys
```
(2).如果要使用其他用户的话，先得将hadoop的目录权限给这个用户添加下
比如：我要使用wxwwt用户来操作
```
useradd wxwwt
passwd wxwwt
chown -R wxwwt:wxwwt /home/hadoop-3.1.2
su wxwwt
```
设置用用户名密码，将hadoop目录权限给wxwwt，然后切换用户。再设置免密登录
```
ssh-keygen -t rsa
ssh-copy-id localhost
```
然后在运行hadoop。

start-hdfs.sh执行完，看下jps如果出现了NameNode，SecondaryNameNode，DataNode的进程就是HDFS启动成功了

#### 5.启动yarn，start-yarn.sh。
启动完后再看下jps，如图出现了ResourceManager，NodeManager就大功告成。
![](https://user-gold-cdn.xitu.io/2019/7/28/16c3905e6c8fe075?w=335&h=124&f=png&s=6359)


### 总结：
1.hadoop的一些简单概念，它也是一个大的生态系统；
2.hadoop安装分三种模式，单机，伪分布式，全分布式；文中介绍的是伪分布式，就是在一台机器上弄的；
3.安装中主要就是按教程添加配置，但是其中还是有坑的，记住启动的时候root的坑，和免密登录。

### 参考资料：
1.https://hadoop.apache.org/docs/r3.1.2/hadoop-project-dist/hadoop-common/SingleCluster.html（官网伪分布式教程）
2.https://blog.csdn.net/solaraceboy/article/details/78438336（免密登录教程）
3.https://blog.csdn.net/whq12789/article/details/79782020
4.https://www.ibm.com/developerworks/cn/opensource/os-cn-hadoop-yarn/（yarn介绍）
