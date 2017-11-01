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
#### 在 Logback 中定义变量
在 `logback.xml` 中添加

```
<define  name="HostName" class="com.lfzhu.logback.LogbackHostName" />

<appender>
    <encoder>
        <pattern>${HostName} - %d{yyyy-MM-dd HH:mm:ss.SSS} %-5level %logger{35} - %msg%n
        </pattern>
    </encoder>
</appender>

```

#### 实现变量取值逻辑

```
public class LogbackHostName extends PropertyDefinerBase {

    @Bean
    public String getPropertyValue() {
        String info = "UnknownHost";
        try {
            Process pro = Runtime.getRuntime().exec("hostname");
            pro.waitFor();
            InputStream in = pro.getInputStream();
            BufferedReader read = new BufferedReader(new InputStreamReader(in));
            info = read.readLine();
        } catch (Exception ex) {
            ex.printStackTrace();
        }
        return info;

    }
}
```

#### Spring 容器预初始化 LogbackHostName 这个 Bean
在 `applicationContext.xml` 中添加

```
<bean id="logbackHostName" class="com.lfzhu.logback.LogbackHostName"/>
```

# 让过滤器(filter)成为一个 Bean
#### web.xml 配置 DelegatingFilterProxy

```
<filter>
    <filter-name>loggingFilter</filter-name>
    <filter-class>org.springframework.web.filter.DelegatingFilterProxy</filter-class>
</filter>

<filter-mapping>
    <filter-name>loggingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

#### Spring 容器预初始化 loggingFilter 这个 Bean

在 `applicationContext.xml` 中添加

```
<bean id="loggingFilter" class="com.lfzhu.common.filter.SystemLoggingFilter"/>
```

(Spring的Bean之Bean的基本概念)[http://www.cnblogs.com/chenssy/archive/2012/11/25/2787710.html]

#### 实现过滤器(filter)逻辑

```
@Service
public class SystemLoggingFilter implements Filter {

    @Autowired
    private KafkaProduceService kafkaProduceService;

    @Override
    public void init(FilterConfig filterConfig) throws ServletException {

    }

    @Override
    public void doFilter(ServletRequest request, ServletResponse response, FilterChain chain)
        throws IOException, ServletException {
        chain.doFilter(request, response);
    }

    @Override
    public void destroy() {

    }

}
```

在这里，我们可以在 `SystemLoggingFilter` 类中执行接口注入的操作(`@Autowired`)。

#### 普通过滤器(filter)的实现方式
在 `web.xml` 中添加

```
<filter>
    <filter-name>loggingFilter</filter-name>
    <filter-class>com.lfzhu.common.filter.SystemLoggingFilter</filter-class>
</filter>

<filter-mapping>
    <filter-name>loggingFilter</filter-name>
    <url-pattern>/*</url-pattern>
</filter-mapping>
```

如果这样配置，我们在 `SystemLoggingFilter` 类中是 ***不能*** 执行接口注入的操作(`@Autowired`)，因为它不是一个 Bean。


