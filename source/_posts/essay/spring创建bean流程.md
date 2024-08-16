---
title: spring创建bean流程
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: essay
---

1.spring创建Bean流程

提前定义好bean的描述信息（xml等文件）
-> 抽象接口 （定义"读取"配置文件的规范，有各种格式比如xml yaml等， 实例 BeanDefinitionReader）
-> Bean的定义信息 （实例 BeanDefintion）
   中间有BeanFactoryPostProcessor：可以用来处理BeanDefinition的信息，比如PlaceHolderPostProcessor，
   来处理占位符的使用，比如${jdbc.username}这样的
-> 实例化Bean
    BeanFactory使用反射创建Bean
    实例化：在堆中给开辟一片空间，属性都是默认值

-> 填充属性


-> 初始化Bean, 执行init方法
    初始化：给属性赋上初始值
    - 分类：
          - 填充属性：赋值操作
          - 调用具体的初始化方法

-> 生成完整的Bean对象


2.AOP
AOP
