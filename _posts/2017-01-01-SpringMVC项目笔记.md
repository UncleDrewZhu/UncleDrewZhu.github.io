---
layout:     post
title:      SpringMVC 项目笔记(持续更新)
subtitle:   针对平时做过的项目记一些笔记
date:       2017-01-01
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Spring
    - 性能测试
    - 日志
    - 过滤器
    - 内存机制
    - JVM
    - 垃圾回收
    - 值传递
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrew.405go.cn/)

# 浅谈 Spring
- Spring Framework(最新版本是5.x，项目上使用的是4.x)
    - IoC(控制反转)
    - DI(依赖注入)
    - AOP(面向切面)
    - MVC
    - Test 
- Spring Data
- Spring Security
- Spring Batch
- Spring Boot


#### Spring 核心思想(IoC, AOP)
###### 前言
一个系统的创建过程就从原先的new改为配置组装，内部通过注入解决了依赖关系，只要满足接口协议即插即用。

###### 引用
> Spring最核心的概念当属IoC, AOP。将对象创建过程的职责赋予容器，通过容器管理对象的生老病死， 
将对象创建过程从编译时延期到运行时，即通过配置进行加载，这样一来就解决了不用编译后期选择具体实现，
其实就是面向对象的核心理念，针对接口编程。
IoC开始就是个factory加上依赖管理罢了，这样一来，一个系统的创建过程就从原先的new改为配置组装，
内部通过注入解决了依赖关系，只要满足接口协议即插即用。
通过IoC, AOP事实上形成了一个套路，通过这个套路完成了系统的整合。


#### Spring 优势
- 模块化开发，逻辑层，表现层，持久层独立分开。
- 通过控制反转降低耦合性，一个对象的依赖通过被动注入的方式而非主动new。
- 面向切面编程，代码不用散布在所有对象层次中，减少重复代码，提高代码复用率。
  
#### Spring AOP
Spring中AOP代理由Spring的IOC容器负责生成、管理，其依赖关系也由IOC容器负责管理。
因此，AOP代理可以直接使用容器中的其它bean实例作为目标，这种关系可由IOC容器的依赖注入提供。
Spring创建代理的规则为：
- 默认使用Java动态代理来创建AOP代理，这样就可以为任何接口实例创建代理了
- 当需要代理的类不是代理接口的时候，Spring会切换为使用CGLIB代理，也可强制使用CGLIB

###### AOP 实现方式
- 使用 @AspectJ 注解
- 在 XML 中配置

###### AOP 实现步骤
1. 定义普通业务组件
2. 定义切入点，一个切入点可能横切多个业务组件
3. 定义增强处理，增强处理就是在AOP框架为普通业务组件织入的处理动作



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

# JVM(Java virtual machine)
JVM 是运行 Java 程序必不可少的机制。JVM 实现了Java 的平台无关性。

- 编译后的 Java 程序指令并不直接在硬件系统的 CPU 上执行，而是由 JVM 执行。
- JVM 屏蔽了与具体平台相关的信息，使Java语言编译程序只需要生成在JVM上运行的目标字节码（.class）,就可以在多种平台上不加修改地运行。
- JVM 在执行字节码时，把字节码解释成具体平台上的机器指令执行。因此实现java平台无关性。
- JVM 是 Java 程序能在多平台间进行无缝移植的可靠保证，同时也是 Java 程序的安全检验引擎（还进行安全检查）。

`JVM = 类加载器 classloader + 执行引擎 execution engine + 运行时数据区域 runtime data area`

***类加载器把硬盘上的 class 文件加载到 JVM 中的运行时数据区域, 由执行引擎负责执行。***



# Java 内存机制

#### 参考

[Java 中堆和栈的区别](http://www.cnblogs.com/perfy/p/3820594.html)

[Java 堆、栈、堆栈的区别](http://www.cnblogs.com/iliuyuet/p/5603618.html)

[Java 内存机制](http://www.cnblogs.com/panxuejun/p/5883264.html)

[Java 内存溢出示例(堆溢出、栈溢出)](http://www.cnblogs.com/panxuejun/p/5882424.html)

[Java 内存泄漏与内存溢出](http://www.cnblogs.com/panxuejun/p/5883044.html)

[Java 内存溢出 栈溢出的原因与排查方法](http://www.cnblogs.com/panxuejun/p/5882309.html)

#### 总结

- 栈(stack)：是一个先进后出的数据结构，通常用于保存方法(函数)中的参数，局部变量。
在 Java 中，所有基本类型(int, short, long, byte, float, double, boolean, char)和引用类型都在栈中存储。
栈中数据的生存空间一般在当前 scopes 内(就是由{...}括起来的区域)。 ***局部变量放在堆栈中。***

- 堆(heap)：是一个可动态申请的内存空间(其记录空闲内存空间的链表由操作系统维护)。
在 Java 中，所有包装类数据(Integer, String, Double)和使用 new xxx() 构造出来的对象都在堆中存储。 ***静态和全局变量，new 得到的变量，都放在堆中。***
当***垃圾回收器***检测到某对象未被引用，则自动销毁该对象。所以，理论上说 Java 中对象的生存空间是没有限制的,只要有引用类型指向它，则它就可以在任意地方被使用。 


# Java 垃圾回收
Java 提供了一种垃圾回收机制，在后台创建一个守护进程。该进程会在内存紧张的时候自动跳出来，把堆空间的垃圾全部进行回收，从而保证程序的正常运行。

能被回收垃圾的定义：
- 创建的对象不再被引用

- 创建的对象不可达

不可回收的对象：
- 虚拟机栈（帧栈中的本地变量表）中引用的对象

- 方法区中静态属性引用的对象

- 方法区中常量引用的对象

- 本地方法栈中 JNI 引用的对象

#### 垃圾回收的方式
1. 标记－清理(容易产生内存碎片，适合存活对象多，垃圾少的情况)
    - 把“存活”对象和“垃圾”对象进行标记
    - 把所有“垃圾”对象所占的空间直接清空
    
2. 标记－整理(不产生内存碎片，适合存活对象多，垃圾少的情况)
    - 把所有存活对象扎堆到同一个地方，让它们待在一起
    
3. 复制(适合存活对象少，垃圾多的情况)
    - 把堆内存分成两部分，一段时间内只允许在其中一块内存上进行分配
    - 当这块内存被分配完后，则执行垃圾回收，把所有存活对象全部复制到另一块内存上
    - 全部清空当前内存



#### 垃圾回收之分代
- 新生代：刚刚创建的对象，存活对象少、垃圾多。

- 老年代：存活了一段时间的对象，存活对象多、垃圾少。

- 永久代：永久存在的对象，比如一些静态文件。

如图：

![](http://oxy6ml8al.bkt.clouddn.com/gc-java-heap-memory.png)

###### 新生代
使用 ***复制*** 回收机制

把内存按 8:1:1分配(可以通过参数 `SurvivorRatio` 手动配置)
- Eden 区：很多新生对象在里面创建
- Survivor A 区：Eden 区经历 GC 后仍然存活下来的对象。
- Survivor B 区：Eden 区和Survivor A 区经历 GC 后仍然存活下来的对象。

当某个 Survivor 区被填满，且仍有对象未被复制完毕时，或者某些对象在反复 Survive 15 次左右时，
则把这部分剩余对象放到 Old 区。

当 Old 区也被填满时，进行 Major GC，对 Old 区进行垃圾回收。


###### 老年代
使用 ***标记整理*** 回收机制

仅仅通过少量地移动对象就能清理垃圾，而且不存在内存碎片化。


#### 参考
[Java 技术之垃圾回收机制](http://www.importnew.com/26821.html)

[JVM 的 工作原理，层次结构 以及 GC工作原理](https://segmentfault.com/a/1190000002579346#articleHeader6)

[深入理解Java虚拟机笔记二（垃圾收集器与内存分配策略）](http://howiefh.github.io/2015/04/08/jvm-note-2/)



# Hibernate 二级缓存

#### 参考
[spring+hibernate 二级缓存 配置+java使用实例](https://www.tuicool.com/articles/qiMBBz)


# Java 里的所有参数都是“值”传递

#### 引用

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

#### 简单的例子

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

# 数据结构
#### HashMap
###### HashMap 的原理
HashMap 是一个用于存储 Key-Value 键值对的集合，每一个键值对也叫做 Entry。这些键值对分散存储在一个数组当中，这个数组就是HashMap的主干。

HashMap 数组每一个元素的初始值都是Null。

(一) Put 方法的原理
以 `hashMap.put("key1", "value1")`为例：

利用哈希函数来确定 Entry 的插入位置（index）： `index =  Hash（"key1"） = 5`

当插入的 Entry 越来越多，Hash 函数会出现 index 冲突的情况。 `index =  Hash（"key2"） = 5`

用 ***链表*** 解决index 冲突：
HashMap 数组的每一个元素不止是一个 Entry 对象，也是一个链表的头节点。每一个 Entry 对象通过 Next 指针指向它的下一个 Entry 节点。
当新来的 Entry 映射到冲突的数组位置时，使用 ***头插法***，将新来的 Entry 插入到对应的链表头，原来位置上的 Entry 在链表上的位置后移。

如图：
![](http://oxy6ml8al.bkt.clouddn.com/hashmap-put.png)

头插法：后插入的 Entry 被查找的可能性更大。


(二) Get 方法的原理
以 `hashMap.get("key1")`为例：

利用哈希函数得到位置(index)： `index =  Hash（"key1"） = 5`

由于同一个位置有可能匹配到多个 Entry，这时候就需要顺着对应链表的头节点，一个一个向下来查找。


###### HashMap 的初始化长度
HashMap 默认的初始化长度是 16，并且每次自动扩展或手动初始化时，长度必须时 2 的幂。

为了实现高效的 Hash 算法，HashMap 采用了位运算的方式。 `index =  HashCode(Key) & (hashMap.Length - 1)`


###### HashMap 在高并发下引起的死锁
在 JDK1.7 环境下，使用HashMap 进行存储时，如果 size 超过当前最大容量*负载因子时候会发生 resize。
其中调用 transfer() 方法时，将每个链表转化到新链表，并且链表中的位置发生反转。
而这在多线程情况下是很容易造成链表回路，从而发生 get() 死循环。


###### HashMap 和 HashTable 的区别
- HashMap 是非 synchronized，而 HashTable 是 synchronized。HashTable 在每个方法调用上加了synchronized。
- HashMap 是线程不安全的，HashTable是线程安全的。
- 单线程环境下，HashMap 速度快。
- HashMap不能保证随着时间的推移 Map 中的元素次序是不变的。

###### 线程安全的 ConcurrentHashMap
- 并发安全
- 直接支持一些原子复合操作
- 支持高并发、读操作完全并行、写操作支持一定程度的并行
- 与同步容器 Collections.synchronizedMap 相比，迭代不用加锁，不会抛出 ConcurrentModificationException
- 弱一致性


###### Java8 对 HashMap 结构的优化 



###### 参考
[什么是HashMap](https://www.itcodemonkey.com/article/1109.html)

[HashMap在高并发下引起的死循环](http://itindex.net/detail/53615-hashmap-%E5%B9%B6%E5%8F%91-%E6%AD%BB%E5%BE%AA%E7%8E%AF)

[HashMap底层实现原理](http://www.cnblogs.com/beatIteWeNerverGiveUp/p/5709841.html)

[Java8系列之重新认识HashMap](http://www.importnew.com/20386.html)
