---
title: maven实战学习笔记一
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: maven
---
### 导语：
上周犯了一个maven的错误，发现自己对maven也只是平时用用没有系统的学习过，所以找了本maven的书系统的看一看。
找了《maven实战》学习一下，总共18章目前看完1-6章，因为maven本来就比较熟悉了，所以笔记只摘录了一些之前没有注意的点。

### 一.编写pom  
1.1maven2，maven3的modelversion只能是4.0.0  
1.2 maven使用groupId artifactId version 三者构成了maven世界的坐标系 能唯一定位一个jar pom war
 而且这三个元素是必须要填写的  
groupId 公司id  
artifaceId 项目id  
version 版本id：  
snapshot：快照版本 不稳定的 可以随时修改的  
releast：发布版本 稳定的 不可修改的  
name：可选参数  用来定义项目的名称  

### 二.编写主代码
1.src/main/java目录用来放Java代码  如果选择了groupId artifactId 包名就是用两者给定义出来的  
2.使用mvn clean compile会将编译好的代码放入到当前目录下的target/下面  
3.执行过程有 clean resource compile 每一个步骤都会对应一个插件  （这种思想
很号模块化 将每个子任务都做成一个插件 没有强耦合 想用哪个插件就配置好流程即可 非常灵活）  
4.dependencies是引入依赖的声明 下面可以放入多个dependency  依赖是根据maven的坐标系 groupId那一套来定义的  
5.dependency中还可以设置scope  比如过是test引入的jar就只会在测试的代码中生效 在main中使用这个test领域的jar就会报错。
默认是主代码（main）和测试代码（test）都生效 (scope这块还是有点绕的需要看很多资料一起看才能明白 我现在也还没完全明白)  
6.maven的compile插件默认只支持1.3  要设置被1.5要额外配置  

### 三.打包和运行  
1.package默认是打成artifact-version.jar 默认是jar的形式  
2.打包的名称也可以指定finalName来命名  
3.compile->test->package->install 每一个后置步骤都会先执行前面的所有步骤  
4.打包完的jar需执行main方法的话要用maven-shade-plugin  指定mainclass  然后就会把mainclass写到mainifest中  

### 四.使用archetype生成项目骨架  
1.archetype有很多种类 比较熟悉的应该就是maven-archetype-quickstart  

### 五.坐标和依赖  
1.基本概念  
1.1classifier：定义附属的一家构件  如：nexus-index-2.0.0-sources.jar sources就是classifier  
1.2命名规则一般是：artifactId-version[-classifier].packaging  
1.3packaging不一定和扩展名向对应，比如maven-plugin对应的也是jar  

2.依赖调解：  
2.1 dependency mediation依赖调解  
第一原则：路径最近者优先  路径最短的jar被使用  
第二原则：第一声明者优先  在pom文件中谁先声明就谁被使用  
备注：1.maven2.0.8之前是不确定的 2.0.9之后 在依赖路径相同的情况下 声明靠前的会被使用  

3.优化依赖：
3.1 查看已解析依赖： mvn dependency:list  
3.2 查看依赖树：mvn dependency:tree  
备注：-DVerbose Verbose not supported since maven-dependency-plugin 3.0  
3.3 分析依赖树：mvn dependency:analyze  
备注： 1.Used undeclared dependencies found  
使用未声明的依赖，没有显示声明但是用到了，存在潜在的风险  
     2.Unused declared dependencies found  
项目中未使用但是显示声明了的依赖 但是analyze只分析了主代码和测试代码 执行测试和运行中的代码是检测不出来的。比如:spring-boot-starter-web这种也会被analyze给分析出来到Unused declared dependencies found但是是不能随便就把这个依赖给删除的，启动项目会报错，analyze分析后的依赖是否删除要具体分析，小心谨慎。  

### 六.仓库
1.仓库种类：  
本地仓库  
远程仓库：  
&nbsp;&nbsp;&nbsp;&nbsp;中央仓库  
&nbsp;&nbsp;&nbsp;&nbsp;其他公共库  
&nbsp;&nbsp;&nbsp;&nbsp;私服  

2.仓库的配置：
respositories的标签下可以写多个repository，但是id必须是唯一的，中央仓库的id默认为central，其它私服仓库如果也用这个名称会覆盖掉中央仓库的配置。
release和snapshots，还有两个子标签，updatePolicy，checksumPolicy。  
updatePolicy：更新策略  
  &nbsp;&nbsp;&nbsp;&nbsp;    always：每次构建都检查更新  
never：从不检查更新  
interval：每隔X分钟检查一次  
daily：默认值是daliy，每天检查一次  

checksumPoliy：检查校验maven的策略  
&nbsp;&nbsp;&nbsp;&nbsp;warn：遇到错误输出警告信息  
fail：遇到错误就构建失败  
ignore:忽略校验和错误  

3.部署到远程仓库  
distributionManagement包含repository和snapshotRepository，用户名密码在maven的settings.xml中配置。  
部署命令：mvn clean deploy  
release和snapshot是基于groupId/artifactId/maven-metadata.xml计算出来的。  

4.镜像  
mirroOf的值如果为central表示所有中央仓库的请求都转发到此仓库，如果是*表示所有的仓库的请求都转发到此仓库。  
```
<mirrorOf>*</mirrorOf>:所有仓库  
<mirrorOf>external:*</mirrorOf>:所有非本机的仓库  
<mirrorOf>repo1,repo2</mirrorOf>:匹配repo1,repo2  
<mirrorOf>*,!repo1</mirrorOf>:匹配所有仓库，除了repo1  
```


### 总结：
1.温习了maven的基础知识，也看到了很多之前没有注意的细节  
2.以后设计系统的时候可以参考maven这种坐标的设计方式，来唯一定位一个对象，然后再加上各种细节配置来完善它  
3.设计插件的时候也可以参考maven的思路，每个插件保持单一职责，可以自由组合使用
