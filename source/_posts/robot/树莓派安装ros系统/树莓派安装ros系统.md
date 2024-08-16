---
title: 树莓派安装ros系统
date: 2023-11-05 16:13:04
updated: 2024-08-15 22:35:58
tags: 树莓派安装ros系统
---
### 导语:
最近给树莓派安装了ros系统，这里记录一下。

### 步骤：
1.下载ros系统的软件  
这里推荐从[ubiquityrobotics](https://downloads.ubiquityrobotics.com)
下载ubiquityrobotics
的系统。这个相当于是给你下载了ubuntu16.04和ros系统一起，只要烧录到树莓派的sd卡里面就可以了,
比较方便。
__注：这里小伙伴跟我说了一个问题，假如是自己下载了ubuntu18的话，然后在手动去安装，ros系统可能会失败，应该是系统版本不兼容的问题。__
不过我没有遇到，比较推荐的还是直接下载上面说的ros的。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/a216d2a936744859bd50e6bbe28d0747~tplv-k3u1fbpfcp-zoom-1.image)


2.烧录系统到SD卡里面  
这里推荐使用balena[](https://www.balena.io/etcher/),选择下载好的系统和要解压的盘，点击运行即可。这个软件还是挺好用的，可以选择本机的文件，也可以选择网络上要下载的文件的地址。而且如果说要你没有选择SD卡，而是错误的选择了自己的系统盘的话，软件会提示你，把本机系统盘给烧录是很危险的，比较人性化。之前也有用过win32diskimager这个软件（不过感觉没这个好用）
。
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/cb850b9023ba452b94163442513fa950~tplv-k3u1fbpfcp-zoom-1.image)

3.烧录号系统后进入系统（使用vnc或者外接屏幕都可以）  
默认的用户名：Unbuntu User
默认的密码：  unbuntu
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/ace1b26eafd54266bb05c169b4c59d29~tplv-k3u1fbpfcp-zoom-1.image)

4.查看ros安装版本   
从左下角的菜单进入System Tools -> LXTeriminal
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/c30b9057d36c4ab29ce6e17d664b8260~tplv-k3u1fbpfcp-zoom-1.image)  

然输入roscore查看安装的ros的版本，到这里基本上就算安装完成了。  
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/0f99de9742a844c9afa8927015ecd213~tplv-k3u1fbpfcp-zoom-1.image)    

不过如果出现了
![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/885428e6166b439dbebe17d4d753e063~tplv-k3u1fbpfcp-zoom-1.image)  
还需要设置下linux的swap区大小，网上说如果小于1G的swap会运行出问题，虽然没有验证过，但是还是设置一下为好。
设置方式可以参考这个网址里面写的[Enabling & Increasing Raspberry Pi Swap](https://nebl.io/neblio-university/enabling-increasing-raspberry-pi-swap/)。
