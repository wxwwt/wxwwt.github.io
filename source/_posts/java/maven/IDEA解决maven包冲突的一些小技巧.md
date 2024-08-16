---
title: IDEA解决maven包冲突的一些小技巧
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: maven
---
&nbsp;&nbsp;&nbsp;&nbsp;在平常工作中我们经常会遇到maven引用的jar包冲突的事情，这时候我们就需要找出冲突的包，并将低版本或者缺少某些方法的jar给剔除掉。这个时候使用idea自带的maven依赖树就很好解决这样的问题。

#### 步骤：
1.在IDEA中右键项项目的pom文件，选择Maven->Show Dependencies，会打开一个maven的依赖树窗口，如下：
![image](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/IDEA%E8%A7%A3%E5%86%B3maven%E5%8C%85%E5%86%B2%E7%AA%81%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%8A%80%E5%B7%A7/show_dependency.png)
2.打开窗口我们可以看到完整的依赖树,小技巧：**左上角有几个小工具，比较常用的1显示冲突项，2显示从root到被选择的jar包路径，3显示实际大小。要选择冲突项的话可以直接点击1，然后在点击3，显示的会更清楚一些，因为jar包比较多，jar依赖比较复杂会让图变得很小。之后如果你需要看这个jar的引用路径可以点击这个jar包再点击2，就回显示从pom文件的根路径的包到被选择的包的单条路线，很方便**；
![img](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/IDEA%E8%A7%A3%E5%86%B3maven%E5%8C%85%E5%86%B2%E7%AA%81%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%8A%80%E5%B7%A7/dependency_mark.png)
3.找到冲突的包后，选择需要的那个jar包，右键要去除的那个jar包，点击exclude，
![img](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/IDEA%E8%A7%A3%E5%86%B3maven%E5%8C%85%E5%86%B2%E7%AA%81%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%8A%80%E5%B7%A7/exclude.png)
就会在pom文件中被剔除（其实就是对应的pom中的exclusion）
![img](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/IDEA%E8%A7%A3%E5%86%B3maven%E5%8C%85%E5%86%B2%E7%AA%81%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%8A%80%E5%B7%A7/pom_exclusion.png)

#### 其他小技巧：
1.在依赖树使用ctrl/command+f是可以直接搜索jar包的名称的；  
![img](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/IDEA%E8%A7%A3%E5%86%B3maven%E5%8C%85%E5%86%B2%E7%AA%81%E7%9A%84%E4%B8%80%E4%BA%9B%E5%B0%8F%E6%8A%80%E5%B7%A7/find_dependency.png)
2.在依赖树的界面使用ctrl/command+鼠标滚轮是可以放大缩小依赖树的比例，同样使用键盘上的+，-号也可以做到这个；  
3.alt/option按住，然后鼠标在依赖树上滑动，是可以达到放大镜的效果的；  
4.在依赖树上双击是可以直接跳转到改jar的引入位置。  
