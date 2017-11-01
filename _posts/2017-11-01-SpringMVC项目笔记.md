---
layout:     post
title:      SpringMVC 项目笔记
subtitle:   针对平时做过的项目记一些笔记
date:       2017-11-01
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Spring
    - 性能测试
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrewzhu.github.io/)

# 使用 PerformanceMonitorInterceptor 进行服务层性能测试
#### AOP 配置切入点
在 `beans-aop-config.xml` 中添加

```
<bean id="performanceMonitor" class="org.springframework.aop.interceptor.PerformanceMonitorInterceptor" />

<aop:config>
    <aop:pointcut id="allServiceMethods" expression="execution(* com.lfzhu.*.service.*.*(..))" />
    <aop:advisor pointcut-ref="allServiceMethods" advice-ref="performanceMonitor" order="1" />
</aop:config>
```

#### Logback 存储性能测试结果
在 `logback.xml` 中添加

```
<property name="USER_HOME" value="logs"/>
<property scope="context" name="FAS_PERF" value="fas-perf"/>
<define  name="HostName" class="com.lfzhu.logback.LogbackHostName" />
<timestamp key="byDay" datePattern="yyyy-MM-dd"/>

<appender name="performance.monitor.file" class="ch.qos.logback.core.rolling.RollingFileAppender">
    <file>${USER_HOME}/${FAS_PERF}.log</file>

    <rollingPolicy class="ch.qos.logback.core.rolling.FixedWindowRollingPolicy">
        <fileNamePattern>${USER_HOME}/${byDay}/${FAS_PERF}-${byDay}-%i.log.zip
        </fileNamePattern>
        <minIndex>1</minIndex>
        <maxIndex>30</maxIndex>
    </rollingPolicy>

    <triggeringPolicy class="ch.qos.logback.core.rolling.SizeBasedTriggeringPolicy">
        <maxFileSize>5MB</maxFileSize>
    </triggeringPolicy>
    <encoder>
        <pattern>${HostName} - %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{35} - %msg%n
        </pattern>
    </encoder>
</appender>

<logger name="org.springframework.aop.interceptor.PerformanceMonitorInterceptor" additivity="false">
    <level value="TRACE"/>
    <appender-ref ref="performance.monitor.file"/>
</logger>

```

#### 性能测试日志展示
在主目录的 `logs` 文件夹下会生成 `fas-perf.log` 文件

```
2016-11-23 15:08:20.221 TRACE o.s.a.i.PerformanceMonitorInterceptor - StopWatch 'com.lfzhu.account.service.UserValidationService.validateUserStatus': running time (millis) = 49
2016-11-23 15:08:20.223 TRACE o.s.a.i.PerformanceMonitorInterceptor - StopWatch 'com.lfzhu.account.service.AppAccountManager.login': running time (millis) = 53
2016-11-23 15:08:20.574 TRACE o.s.a.i.PerformanceMonitorInterceptor - StopWatch 'com.lfzhu.kafka.service.KafkaProduceService.sendMessageToKafka': running time (millis) = 128
```

文件中的每一条日志表示一个接口执行的速度，我们可以针对速度慢的服务进行性能优化。

# Logback 生成日志时标示宿主主机信息
