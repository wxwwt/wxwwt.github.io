---
title: maven导入snapshot依赖找不到对应类
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: maven
---
maven导入snapshot依赖找不到对应类



### 导语：
最近在做项目的时候，引入公司编写的二方库的包，maven仓库也deploy上去了。然后编译代码的时候发现一直找不到一个类，就编译不通过。
一开始以为是本地idea或者maven的缓存导致没有拉取到最新的包。后来经过一系列的无用操作，发现了问题的所在是二方库的snapshot生成有个时间戳导致找不到。
下面来看一下解决的过程：

### 解决步骤：
1.首先确认了二方库的版本和项目引入的二方库的版本是不是一致的，结果发现版本是没有问题的  
2.确认二方库的包是不是真的传到maven仓库了，直接从maven的私服去搜索这个jar包发现这个包是存在的，
而且有很多版本，找到了最新版本的那个jar下载下来，使用jd-gui反编译之后发现类也是存在的  
3.确认是不是idea或者maven的缓存导致使用的一直是原来的二方库jar包？清理了所有缓存之后，依然编译的时候找不到对应的类  
4.经过以上的三个步骤依然没有解决当前的问题，但是想到引用的第三方的jar并没有出现这样的问题，那会不会是二方库的问题。  
在把项目打包之后，把打包完成的jar用反编译工具解开。
里面大概会有这么几个文件META-INF，项目包文件。然后点开META-INF里面有一个MAINIFEST.MF,里面是编译的配置信息。
然后有个叫Class-Path的参数，里面写的是编译引入的jar的路径。仔细研究发现这个引入的二方库的包在class-path的路径里面是
xxx-时间戳.jar的这么一个格式。其他的jar包都是很正常的名字.jar的格式，所以很有可能是这个名字有个时间戳的原因导致的。
查看资料发现，确实是这样的情况，class-path的路径里面写的是时间戳，但是打包的文件里面对应的jar是-snapshot结尾的名字。
所以直接根据mainifest的名字去找对应的jar是找不到的，所以就出现了明明已经引入了jar包但是却编译的时候显示找不到类的问题。

**解决办法就是在maven的打包插件里面加一个useUniqueVersions参数**，意思是是否使用唯一的版本号，默认值是true，如果是带时间戳的这种就会
每一个时间戳都是一个版本。如果改为false，就会把后面的时间戳给转为-snapshot这种正常格式的。这样mainifest里面class-path和包名就能统一了。

```
<plugin>
	<groupId>org.apache.maven.plugins</groupId>
	<artifactId>maven-jar-plugin</artifactId>
		<configuration>
			<archive>
				<manifest>
					<addClasspath>true</addClasspath>
					<classpathPrefix>lib/</classpathPrefix>
					<mainClass>com.xxx.Application</mainClass>
					<useUniqueVersions>false</useUniqueVersions>
				</manifest>
			</archive>
		</configuration>
</plugin>
```

这里附上maven官网对于这个参数的说明，[参考链接](http://maven.apache.org/shared/maven-archiver/index.html)
。搜索这个问题的时候发现之前maven的官网在12年就已经有人提出过这个问题了，[问题链接](https://issues.apache.org/jira/browse/MJAR-156)。

### 总结：
有些东西平时没怎么注意，只理解了怎么使用，在某些时候就可能会耽误大量的时间去解决。这个问题我弄了一天半才解决，之前一直以为是maven仓库或者idaea的缓存问题导致的。这种问题也不明显，因为确实看到了就是二方库的jar包已经打进去了，类文件也是存在的。感觉对maven这一块其实并不是太了解，主要停留在使用层面上，
这种问题要能迅速解决的话，一个是自己对maven非常的熟悉，每个插件的用法，每个参数的含义，都很清楚一看就知道是什么问题了，另一个就是积累经验了，多做记录以后再出现了就能知道是什么问题。

举一反三的话，就是解决问题分为主动的解决和被动的解决，主动就是在还没有出现问题的时候做好预案，对事物有了充分的认识和理解，一旦问题出现就能立马搞定。
被动解决就是出现了问题再去了解问题是怎么产生的，改怎么处理。很明显主动要优于被动，所以在时间充裕或者说自己挤出时间多主动去学习，记录，输出才能成为大牛。

### 参考资料：  
1.[maven官网问题列表1](https://issues.apache.org/jira/browse/MJAR-156)  
2.[maven官网问题列表2](https://issues.apache.org/jira/browse/MSHARED-169)  
3.[maven snapshot的jar包中的类找不到](https://www.javatt.com/p/81337)  
4.[maven官网useUniqueVersions的说明](http://maven.apache.org/shared/maven-archiver/index.html)
