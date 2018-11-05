---
layout:     post
title:      SpringBoot 项目笔记
subtitle:   记录一些自己对 SpringBoot 的认知和总结
date:       2018-06-25
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - SpringBoot
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrew.405go.cn/)


# SpringBoot
- 使编码变简单
- 使配置变简单
- 使部署变简单
- 使监控变简单

# AutoConfiguration 的原理
- ConditionalOnClass
- ConditionalOnMissingClass
- ConditionalOnBean
- ConditionalOnMissingBean

# Maven
## scope

```
<!--编译需要而发布不需要的jar包-->
<dependency>
    <groupId>javax.servlet.jsp</groupId>
    <artifactId>jsp-api</artifactId>
    <version>2.1</version>
    <scope>provided</scope>
    <classifier />
</dependency>
```

- compile：默认的scope，表示 dependency 都可以在生命周期中使用。而且，这些dependencies 会传递到依赖的项目中。适用于所有阶段，会随着项目一起发布
- provided：跟compile相似，但是表明了dependency 由JDK或者容器提供，例如Servlet AP和一些Java EE APIs。这个scope 只能作用在编译和测试时，同时没有传递性。
- runtime：表示dependency不作用在编译时，但会作用在运行和测试时，如JDBC驱动，适用运行和测试阶段。
- test：表示dependency作用在测试时，不作用在运行时。 只在测试时使用，用于编译和运行测试代码。不会随项目发布。
- system：跟provided 相似，但是在系统中要以外部JAR包的形式提供，maven不会在repository查找它。

