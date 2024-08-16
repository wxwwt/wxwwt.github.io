---
title: Langchain-Chatchat安装使用
date: 2024-03-11 22:57:19
updated: 2024-08-15 22:35:58
tags: ai
---
Langchain-Chatchat安装使用

# 1.前言：

最近AI爆发式的火，忆往昔尤记得16,17那会移动互联网是特别火热的，也造富了一批公司和个人，出来了很多精妙的app应用。现在轮到AI发力了，想想自己也应该参与到这场时代的浪潮之中，所以就找了开源的项目来玩一玩，学习下里面的知识。不管最后结果有没有造富自己，学到的知识总是有用的，至少不会让自己在AI时代掉队。

今天要讲的是LangChain-chatchat， 用官网自己的话来说就是： 基于 Langchain 与 ChatGLM 等大语言模型的本地知识库问答应用实现。 一种利用 [langchain](https://github.com/hwchase17/langchain) 思想实现的基于本地知识库的问答应用，目标期望建立一套对中文场景与开源模型支持友好、可离线运行的知识库问答解决方案。

界面如下：

![image-20240310194452916](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310194452916.png)

# 2.安装步骤：

官网有三种安装方式：

1.autoDL

2.docker

3.本地部署

第一种其实比较方便和实惠，机器配置不够也可以跑模型，每个小时几块钱，而且都是autoDL有对应的镜像可以直接运行，非常的便捷。

第二种大概有40G的包，部署也算比较方便。

今天我们讲的是第三种本地部署，虽然比较麻烦，但是在自己机器上部署方便调试，也更容易去了解整个项目是怎么运行的，对于学习来说是比较好的。

## 2.1 前置条件

### 硬件：

- 官网推荐：

  - 本框架使用 `fschat`驱动，统一使用 `huggingface`进行推理，其他推理方式(如 `llama-cpp`，`TensorRT加速引擎` 建议通过推理引擎以 API 形式接入我们的框架)。

    同时, 我们没有对 `Int4` 模型进行适配，不保证`Int4`模型能够正常运行。因此，量化版本暂时需要由开发者自行适配, 我们可能在未来放。

    如果想要顺利在GPU运行本地模型的 **FP16** 版本，你至少需要以下的硬件配置，来保证在我们框架下能够实现 **稳定连续对话**

    - ChatGLM3-6B & LLaMA-7B-Chat 等 7B模型
      - 最低显存要求: 14GB
      - 推荐显卡: RTX 4080
    - Qwen-14B-Chat 等 14B模型
      - 最低显存要求: 30GB
      - 推荐显卡: V100
    - Yi-34B-Chat 等 34B模型
      - 最低显存要求: 69GB
      - 推荐显卡: A100
    - Qwen-72B-Chat 等 72B模型
      - 最低显存要求: 145GB
      - 推荐显卡：多卡 A100 以上

    一种简单的估算方式为：

    ```
    FP16: 显存占用(GB) = 模型量级 x 2
    Int4: 显存占用(GB) = 模型量级 x 0.75
    ```

    

    以上数据仅为估算，实际情况以 **nvidia-smi** 占用为准。 请注意，如果使用最低配置，仅能保证代码能够运行，但运行速度较慢，体验不佳。

    同时，Embedding 模型将会占用 1-2G 的显存，历史记录最多会占用 数GB 的显存，因此，需要多冗余一些显存。

    内存最低要求: 内存要求至少应该比模型运行的显存大。

    例如，运行ChatGLM3-6B `FP16` 模型，显存占用13G，推荐使用16G以上内存。

    ### 部分测试用机配置参考，在以下机器下开发组成员已经进行原生模拟测试（创建新环境并根据要求下载后运行），确保能流畅运行全部功能的代码框架。

    

    - 服务器

    ```
    处理器: Intel® Xeon® Platinum 8558P Processor (260M Cache, 2.7 GHz)
    内存: 4 TB
    显卡组:  NVIDIA H800 SXM5 80GB x 8
    硬盘: 6 PB 
    操作系统: Ubuntu 22.04 LTS,Linux kernel 5.15.0-60-generic
    显卡驱动版本: 535.129.03
    Cuda版本: 12.1 
    Python版本: 3.11.7
    网络IP地址：美国，洛杉矶
    ```

    

    - 个人PC

    ```
    处理器: Intel® Core™ i9 processor 14900K 
    内存: 256 GB DDR5
    显卡组:  NVIDIA RTX4090 X 1 / NVIDIA RTXA6000 X 1
    硬盘: 1 TB
    操作系统: Ubuntu 22.04 LTS / Arch Linux, Linux Kernel 6.6.7
    显卡驱动版本: 545.29.06
    Cuda版本: 12.3 Update 1
    Python版本: 3.11.7
    网络IP地址：中国，上海 
    ```

- 我的电脑：

  ```
  处理器: 13th Gen Intel(R) Core(TM) i5-13490F
  内存: 32GB DDR5
  显卡组:  NVIDIA RTX4060
  硬盘: 2TB
  操作系统: windows wsl2安装的Ubuntu 22.04.3 LTS
  显卡驱动版本: 536.67
  Cuda版本: 12.2
  Python版本: 3.10.12
  ```

  

### 软件：

- 官网推荐：

  要顺利运行本代码，请按照以下系统要求进行配置

  **已经测试过的系统**

  - Linux Ubuntu 22.04.5 kernel version 6.7

  其他系统可能出现系统兼容性问题。

  **最低要求**

  该要求仅针对标准模式，轻量模式使用在线模型，不需要安装torch等库，也不需要显卡即可运行。

  - Python 版本: >= 3.8(很不稳定), < 3.12
  - CUDA 版本: >= 12.1

  **推荐要求**

  开发者在以下环境下进行代码调试，在该环境下能够避免最多环境问题。

  - Python 版本 == 3.11.7

  - CUDA 版本: == 12.1

    

- 笔者电脑：

  - 系统：windows wsl2安装的Ubuntu 22.04.3 LTS
  - python版本:  3.10.12
  - CUDA版本：12.2



之所以提一下电脑硬件软件的配置，因为可能存在刚好有读者跟我的差不太多的硬件配置，或者比我好的硬件配置就是可以跑起来的。而且软件这个我可以跑起来的话，也验证了在我这个系统，python版本，CUDA版本的组合是可以运行起来的，也可以给别人一个参考。要注意一点的就是，如果你跟我一样的是使用windows的系统，然后wsl走的linux系统，提一嘴就是windows上安装的cuda版本可能会跟linux系统的cuda版本不一样的情况，最后是卸载掉弄成一样的，小于11.7的话跑通义千问的模型会有问题，虽然我跑通义千问的模型还没有成功，但是在解决一个安装qwen模型的时候遇到一个问题就是安装某个依赖库的时候要求cuda版本大于11.7。



## 2.2 部署步骤

- 拉取代码

  ```
  # 拉取仓库
  $ git clone --recursive https://github.com/chatchat-space/Langchain-Chatchat.git
  
  # 进入目录
  $ cd Langchain-Chatchat
  
  # 安装全部依赖
  $ pip install -r requirements.txt
  ```

- 下载模型

  ```
  # 安装模型，这一步如果没有进行，启动项目的时候回自动从https://huggingface.co/上面下载，不过问题就是
  # 国内从https://huggingface.co/上下载模型是很慢的。所以建议先从modelscope（魔塔上下载模型），然后在
  # 项目的configs/model_config.py填写好MODEL_ROOT_PATH地址，这样不用从外部下载模型直接跑对于第一次运
  # 行会快很多。 
  
  # 下载模型,下载模型需要先安装Git LFS，然后运行。官网使用的是https://huggingface.co的包，我这里修
  # 改成魔塔的仓库地址了。不过要提一点的是虽然官网要你下载了这两个模型，如果没有修改配置文件里面的话，跑
  # 起来用的并不这两个模型。chatglm2-6b这个是llm（大语言模型），m3e-base这个是embeding模型。需要在
  
  $ git lfs install
  $ git clone https://www.modelscope.cn/ZhipuAI/chatglm2-6b.git
  $ git clone https://www.modelscope.cn/Jerry0/m3e-base.git
  ```

  - 配置模型

  将项目中configs/model_config.py里面的LLM_MODELS里面增加上chatglm2-6b，EMBEDDING_MODEL配置上m3e-base。配置完之后，才会在启动的时候使用下载的这两个模型，要不然会使用默认的模型。chatglm3-6b和bge-large-zh-v1.5。

  ![image-20240310193905361](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310193905361.png)

  

  tips：因为上面说的这两个模型是已经有开发者验证过的所以在下面的模型列表里面是有的，下载完模型，修改下配置文件就可以用。但是如果模型列表里面没有的模型加载进来，不一定可以跑。这个要注意下！

![image-20240310193814840](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310193814840.png)

- 启动项目

  如果什么问题到没有出现的话，就会是这样一个界面，上面会显示加载的LLM模型，使用的Embedings模型，项目api文档地址和webui的地址。

  ![image-20240310194211001](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310194211001.png)

​       可以看到我们可以访问本地的8501端口就可以进入到web界面。

​	![image-20240310194452916](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310194452916.png)

- 实践效果

使用本地机器跑模型的效果，虽然这个图里面是回答得感觉还行吧，但是实际我问一个问题，回答需要可能10分钟才能返回完结果。可能是因为这个确实挺需要硬件资源的，我本地就一块显存8G的显卡，能跑起来，我已经是谢天谢地了。而且我还找了些资料去优化，将FP16的模型弄成int8的模型去跑，但是不知道是我方式不对还是，硬件资源不够，跑出来的效果也还是很慢。所以如果想要商业化之类的，硬件资源还是得给够，或者走大模型的api调用。我这个只能说是个人学习使用下，连流畅的效果都达不到，哈哈哈。

![image-20240310181143407](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310181143407.png)

# 3.遇到的问题

3.1 python安装库特别慢，查了一下，如果运行 `pip config list` 返回空值，表示没有明确在配置文件中设置源地址。在这种情况下，pip将使用其内置的默认源，即 https://pypi.org/simple。ping了一下这个地址，时延有几百毫秒，而且丢包严重。后来查了下资料换成了清华的源，

设置步骤如下：

```
# 在命令行输入
pip config set global.index-url https://pypi.tuna.tsinghua.edu.cn/simple
```



3.2 下载模型的时候git clone连不上服务器

方法一：

发现访问这个模型需求一些科学的手段，直接浏览器可以访问到https://huggingface.co/THUDM/chatglm3-6b的模型，但是git不行，于是使用了临时代理。

设置如下：

```
# 在命令行输入
 git clone -c http.proxy="http://127.0.0.1:1001" https://huggingface.co/THUDM/chatglm2-6b
 
 git clone -c http.proxy="http://127.0.0.1:1001" https://huggingface.co/BAAI/bge-large-zh
```

这个代理的端口就用自己平时完成科学访问的端口。

方法二(推荐)：

另外一种方式就是访问国内的魔塔网站下载（modelscope），进入到模型库的栏目，

搜索对应模型，然后点击下载。以chatglm2-6b为例。

```
git clone https://www.modelscope.cn/ZhipuAI/chatglm2-6b.git
```

因为是国内直接下载，方法二比方法一还是快很多，推荐使用这个方式。



3.3 报错ModuleNotFoundError: No module named 'pwd'

这个报错来自于我一开始是在windows系统上部署的，发现官方推荐的系统是ubuntu，我本地是用的windows系统，执行启动脚本的时候需要使用到linux的pwd的命令。windows里面是没有这个命令的所以报错了，本来想改写下这个脚本使用windows对应pwd的命令去处理。但是一想，万一后面还有其他地方也用了只有linux存在的命令，那改起来的地方就多了，还是老老实实的用linux系统吧。所以后来重新弄了下windows11的WSL，用WSL可以在windows系统下安装linux的子系统，然后让Chatchat在linux子系统里面跑应该就没问题了。

```
# 在windows11命令行执行，如果没有安装过这个，可以自己看下最下面的参考资料有提到怎么在windows11上开启wsl，主要是有一些虚拟机开关要打开。
wsl --install
```

这里简单提一下默认的ubuntu的目录和windows的系统磁盘的对应关系，在unbuntu里面进入到/mnt目录，然后比如你要进入win的d盘，就输入cd /mnt/d就行了。其他盘符也是一样的道理。



3.4 安装qwen的模型报错

![image-20240309145022872](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240309145022872.png)

这个问题来自于准备使用通义千问的模型，然后需要启动chatchat提示需要安装一个fast-attention的包，上面这个图就是安装fast-attention报的错。因为我本地的cuda是没有加入到环境变量里面的，所以报错了。后来我下载了一个cuda11.5，结果继续报错，查资料说是要11.7以上。后来又卸载了，更新成cuda12.2才可以。

3.5 register_controller报错

![image-20240310195600397](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310195600397.png)

这个报错是提示register_to_controller报错，是问了交流群里面的人才解决的，是因为我本地起了全局代理，然后这个是注册应该走到代理的网络上去了。关闭了代理或者PAC模式之后，wsl要重新启动一个新的会话，然后再启动项目就可以运行了。

3.6 chatchat开启量化模型

这个问题是来源于我感觉本地的llm返回很慢，所以查了下资料怎么优化返回速度。发现可以修改项目中configs/server_config.py里面的load_8bit参数。

![image-20240310200634568](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310200634568.png)

开启之后，启动项目的加载模型的日志里面会多一行'load_8bit': True的日志，表示开启8bit量化成功。这个原理大概是这样一个意思，本身模型的计算可能小数位很长，假设有16位，开启之后把16位转为8位或者精度更低的位数，这样计算的时候就会更加迅速，不过带来的问题就是可能结果没有之前准确。不过我试了下开启之后，我主观上没有觉得它返回变快了，不过群里的朋友说开启之后是挺快的，这个效果我是没有办法百分之百说有效，读者可以自行尝试一下。

# 4.项目结构

自己理解的项目结构，可能不完全准确

![image-20240311080104198](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240311080104198.png)

官网放的Chatchat处理流程图，如果看过langchain的资料的话，会发现中间主要是langchain的处理过程，因为这个项目也是基于langchain去做的。

![image-20240310202107156](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310202107156.png)

文档处理流程

![image-20240310202128044](https://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/imgRepo/image-20240310202128044.png)



# 5.总结

虽然使用Chatchat整个过程中的坑还是挺多的，但是至少跑起来了，而且在跑这个项目中遇到了很多自己没有接触过的知识。比如量化模型这个概念，是在优化返回速度的时候才知道可以把模型的精度改小，提高计算速度。现在本地还只运行成功了项目本身支持的几个模型，像界面中的知识库问答，文件对话，搜索引擎问答，自定义agent都还没跑成功，还有挺多东西要去研究和尝试的，还是挺有意思的。我想了想后面可能会针对其他的几个模式也写一些记录。



# 6.参考资料：

[1.如何使用 WSL 在 Windows 上安装 Linux](https://learn.microsoft.com/zh-cn/windows/wsl/install)

[2.本地安装部署运行 ChatGLM-6B 的常见问题解答以及后续优化](https://www.tjsky.net/tutorial/667)

[3.LangChain-Chatcaht项目地址](https://github.com/chatchat-space/Langchain-Chatchat)