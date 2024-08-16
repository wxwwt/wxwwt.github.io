---
title: 用Typora，PicGo和OSS实现自动上传图片
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: essay
---


# 前言：

以前写博客要发布到好些个平台，我是将图片一张张上传到每个平台，后来发现是真的麻烦，上传图片花的时间太多，极大的降低了我写文章的积极性。

后来改进为使用oss，把博客的图片都上传到oss上面。然后使用oss返回的图片url，这样我的文章里面的图片只上传了一次，最后把整篇文章的mardown复制到各个平台上，平台一般都会把markdown的文章中的img标签的图片上传到他们自己的服务器，然后把图片打上平台的水印，然后把原图片链接替换掉。这样图片值上传了一次，剩下平台上的图片都是平台解析markdown的图片url自己处理的，不需要我们在花时间去上传图片了。

写了一段时间，还是觉得上传一次都觉得是在浪费生命（科技是第一生产力，但是懒惰是第一需求力啊），于是又找了找，还有更加方便的方式吗？我直接截图就自动上传，不用我再传一遍的省力方式？

最后是使用typora+picgo+oss解决了问题。

简单说一下我尝试的其他方式的过程，首先我们用的编辑文章的编辑器要支持直接放截图，并且能上传截图，找了找发现以前用过的typora是完美的符合需求的编辑器，但是发现以前的版本比较低，去官网重新下载了最新版本，结果令我很意外的是以前免费的typora竟然收费了！看来这几年大家都越来越会搞钱了，不过这个编辑器确实还是比较好用的，还是应该支持下优秀的产品，经济水平不错的同学可以支持下，说来惭愧，我是用脚本白嫖的，手动狗头，别打我。写到这里，我突然想起刘强东说kindle在中国肯定做不下去，因为中国充斥着大量的盗版软件，他说的没错，kindle在22年6月30号停止了电子书的运营。或许免费使用，再对高级功能进行收费比较适合我国国情，免费使用用来提高产品的影响力和吸引用户，在对愿意支持收费的用户提供高级功能和收费的入口，这样应该就可以解决产品的盈利问题了。

稍微扯了扯题外话，继续聊自动上传图片的话题，我一开始想能不能直接用github，这样我连钱都不用花，岂不快哉！结果按照网上的教程尝试过后发现，如果不能掌握上网的正确方式，现在的情况是github都连不上，更不用说用github做图床了。然后想试下国内的gitee，结果试了半天发现gitee现在上了防盗链，如果其他人访问的话，没有gitee的域名图片肯定是无法访问的，这也走不通。

最后还是回到我之前的图床-oss，我买的好像是49块钱的，用了好几年也没收过其他的费用。然后使用picgo作为typora上传图片的工具，就搞定了，下面记录一下，搭建自动上传图片工具的过程。



# 搭建流程

## 1.先准备一个工具typora，和购买阿里云oss的服务
[typora官网的下载地址](https://typoraio.cn/)   
笔者这里使用的是typora1.4.8版本

## 2.设置typora，进入偏好设置

![image-20230102164627305](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102164627305.png)

## 3.选择【图像】的设置，插入图片选择上传图片。

设置如下，选择下面几个跟图片相关的配置：


![image-20230102152039088](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102152039088.png)

## 4.【上传服务器设定】，选择【PicGo(app)】

如果本地没有下载过PicGo，也可以直接点击上面的图中的【下载PicGo（app）】，会下载PicGo的应用。当前我们也可以不选择PicGo选择其他的插件，我是觉得PicGo有图形化界面比较好用，而且还可以脱离typora单独使用，还是国人写的，就选择了这个插件，下面放一下它的官网链接。

[PicGo的官网](https://picgo.github.io/PicGo-Doc/zh/guide/)

然后下载好PicGo之后，要在typora里面配置好PicGo的运行路径，像上面的图中那样，指定好运行的目录。

**tips:这里稍微扯两句，如果不需要上传到云上，就把图片保存到本地，可以选择复制图片到当前的文件夹，或者下面两个复制到XXX文件夹。这样图片可以直接保存在本地，也很方便。**

![image-20230102164810848](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102164810848.png)



## 5.oss相关配置（如果自己已经买好oss服务的同学可以忽略第5步）

这里分成两个步骤：

如果直接用自己的主账号的话，就直接前往5.5步。

如果为了更安全一点，创建一个RAM子账号专门来管理oss的事情，防止其他不必要的信息暴露，创建RAM子账号，就在这里继续往下看。

### 5.1 从右上角的下拉菜单里面点击【访问控制】

![image-20230102172610021](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102172610021.png)

### 5.2 在点击【身份管理】的【用户】，在点击右边的【创建用户】

![image-20230102173136573](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102173136573.png)

###  5.3 创建用户记得勾选【OpenApi调用访问】
这个是PicGo上传图片要用的

![image-20230102173820775](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102173820775.png)

### 5.4 记住要给设置的用户添加权限【AliyunOSSFulAccess】

**这个权限给Ram用户一定要设置下，要不然没法使用oss的各种操作**

![image-20230102174319723](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102174319723.png)

### 5.5 找到账号对应的key和Secret
复制好key和secret，后面要用
![image-20230102201534323](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102201534323.png) 



### 5.6 创建Bucket

Bucket可以理解为创建一个放图片的空间，里面还可以在继续设置文件夹。

**tips：要记住设置公共读，因为我们的图片是要让大家能在平台上传时能访问的。**

另外解释一下：

【私有】就是上传和访问都是需要令牌的

【公共读】就是上传图片是需要令牌（appKey和secret），访问是大家都可以访问的

【公共读写】就是不管是谁，都可以上传和访问

![image-20230102175605624](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102175605624.png)

![image-20230102145523507](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102145523507.png)

### 5.7 可以给要存储图片的目录创建一个文件夹（可选，不创建也没有关系）

点击刚才创建好的Bucket，为了方便我们管理，右边选择新建目录，比如picture之类的。







## 6.配置PicGo

在【图床设置】里面选择【阿里云OSS】，填入之前复制的key和Secret，还有bucket，储存区域可以从bucket点开，【概况】里面可以看到，.aliyuncs.com之前的oss-cn-hangzhou就是。

![image-20230102202108502](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102202108502.png)

![image-20230102195958205](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102195958205.png)

## 7.验证下效果

使用截图工具随便截个图，粘贴到typora中，看到上传成功，就大功告成！

这篇文章里面的图片也全是用这个方式上传的，再也不用自己手动上传图片了，哈哈哈，节省了自己不少时间和精力，还是挺好的!
![image-20230102202514623](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20230102202514623.png)



# 参考资料：

[1.Typora使用技巧之插入图片及图片上传](https://zhuanlan.zhihu.com/p/344941041)  
[2.阿里云 OSS + PicGo 博客图床超详细配置教程！  ](https://dlonng.com/posts/ailyun-oss)
[3.Typora ](https://typoraio.cn/) 
[4.PicGo ](https://picgo.github.io/PicGo-Doc/zh/) 
[5.Typora详解 ](https://juejin.cn/post/7067166508178210846) 
