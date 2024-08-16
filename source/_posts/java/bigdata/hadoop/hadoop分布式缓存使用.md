---
title: hadoop分布式缓存使用
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: hadoop
---
### hadoop分布式缓存的使用

### 介绍
DistributedCache是hadoop框架提供的一种机制,可以将job指定的文件,在job执行前,先行分发到task执行的机器上,并有相关机制对cache文件进行管理。
缓存内容是在文件中的，各个节点可以根据hdfs中访问路径来读取缓存。

### 使用步骤
1.添加分布式缓存的时候，
先定义缓存的路径
```
String cacheFile = "hdfs://xxxx";
```
可以设置别名 “#”号后面的就是别名 在方法中可以直接使用
cacheFile = cacheFile + “#别名”
在main方法中添加到job中（然后在map阶段就可以使用）
```java
// 缓存jar包到task运行节点的classpath中
job.addArchiveToClassPath(archive);
// 缓存普通文件到task运行节点的classpath中
job.addFileToClassPath(file);
// 缓存压缩包文件到task运行节点的工作目录
job.addCacheArchive(uri);
// 缓存普通文件到task运行节点的工作目录
job.addCacheFile(uri)
```
2.使用分布式缓存
```java
  @Override
  protected void setup(Mapper<LongWritable, Text, Text, Text>.Context context) throws IOException, InterruptedException {
      super.setup(context);
      // 读取缓存文件中的内容 直接根据别名读取
          FileReader fr = new FileReader("别名");
          BufferedReader br = new BufferedReader(fr)；
  }
```
