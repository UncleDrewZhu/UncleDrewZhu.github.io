---
layout:     post
title:      SpringMVC 项目笔记(持续更新)
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


# Java 内存机制

[Java 中堆和栈的区别](http://www.cnblogs.com/perfy/p/3820594.html)

[Java 堆、栈、堆栈的区别](http://www.cnblogs.com/iliuyuet/p/5603618.html)

[Java 内存机制](http://www.cnblogs.com/panxuejun/p/5883264.html)

[Java 内存溢出示例(堆溢出、栈溢出)](http://www.cnblogs.com/panxuejun/p/5882424.html)

[Java 内存泄漏与内存溢出](http://www.cnblogs.com/panxuejun/p/5883044.html)

[Java 内存溢出 栈溢出的原因与排查方法](http://www.cnblogs.com/panxuejun/p/5882309.html)


- 栈(stack)：是一个先进后出的数据结构，通常用于保存方法(函数)中的参数，局部变量。
在 Java 中，所有基本类型(int, short, long, byte, float, double, boolean, char)和引用类型都在栈中存储。
栈中数据的生存空间一般在当前 scopes 内(就是由{...}括起来的区域)。 ***局部变量放在堆栈中。***

- 堆(heap)：是一个可动态申请的内存空间(其记录空闲内存空间的链表由操作系统维护)。
在 Java 中，所有包装类数据(Integer, String, Double)和使用 new xxx() 构造出来的对象都在堆中存储。 ***静态和全局变量，new 得到的变量，都放在堆中。***
当垃圾回收器检测到某对象未被引用，则自动销毁该对象。所以，理论上说 Java 中对象的生存空间是没有限制的,只要有引用类型指向它，则它就可以在任意地方被使用。 

# Java 里的所有参数都是“值”传递

> 其实Java里的所有参数都是“值”传递。之所以用引号括起来是因为值传递容易让人产生误解。
八个基础类型是按照一般人理解的值传递的方式传递的，将变量对应的值复制一份传递给函数。
而其他所有的引用类型依然是按照“值”传递的方式传递的，但是他们传的值不是对象的一份值拷贝，
而是指向对象的引用的一份值拷贝，因此你可以在方法内通过传入的对象引用的值拷贝操作外界对象，
但是一旦重新让这个引用的值拷贝指向别的对象，那么接下来的一切就跟外部对象没有关系了。
  
> Java程序设计语言总是采用值调用。也就是说，方法得到的是所有参数值的一个拷贝，
特别是，方法不能修改传递给它的任何参数变量的内容。

> 在Java中，变量分为以下两类：
①对于基本数据类型变量（int、long、double、float、byte、boolean、char），Java是传值的副本。
②对于一切对象型变量，Java都是传引用的副本。
其实传引用副本的实质就是复制指向地址的指针，只不过Java不像C++中有显著的*和&符号。
需要注意的是：String类型也是对象型变量，所以它必然是传引用副本。
String类是final类型的，因此不可以继承和修改这个类。

概念:
- 值传递：表示方法接收的是调用者提供的值。

- 引用传递：表示方法接收的是调用者提供的变量地址。

Java 中参数传递情况如下： 
- 一个方法不能修改一个基本数据类型的参数

- 一个方法可以修改一个对象参数的状态

- 一个方法不能实现让对象参数引用一个新对象

```
    public static void main(String[] args) {
        Employee employee1 = new Employee();
        employee1.age = 100;
        changeEmployee1(employee1);
        System.out.println(employee1.age);//100

        Employee employee2 = new Employee();
        employee2.age = 100;
        changeEmployee2(employee2);
        System.out.println(employee2.age);//1000

        String x1 = new String("ab");
        change1(x1);
        System.out.println(x1);//ab

        String x2 = new String("ab");
        change2(x2);
        System.out.println(x2);//ab
    }

    private static void changeEmployee1(Employee employee) {
        employee = new Employee();
        employee.age = 1000;
    }

    private static void changeEmployee2(Employee employee) {
        employee.age = 1000;
        employee = new Employee();
        employee.age = 2000;
    }

    //传过去的参数实际拷贝了一份，刚开始一同指向"ab"，后来指向“cd”就跟原来x的没什么关系了。
    private static void change1(String x) {
        x = "cd";
    }

    private static void change2(String x) {
        x += "cd";
    }
```
