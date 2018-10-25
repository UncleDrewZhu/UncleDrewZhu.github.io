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
