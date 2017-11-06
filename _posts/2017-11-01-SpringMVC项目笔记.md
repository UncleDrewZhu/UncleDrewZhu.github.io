---
layout:     post
title:      SpringMVC 项目笔记(持续更新)
subtitle:   针对平时做过的项目记一些笔记
date:       2017-11-01
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - 性能测试
    - 日志
    - 过滤器
    - 内存机制
    - 值传递
    - 依赖管理
    - 多模块
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrew.405go.cn/)

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
当垃圾回收器检测到某对象未被引用，则自动销毁该对象。所以，理论上说 Java 中对象的生存空间是没有限制的,只要有引用类型指向它，则它就可以在任意地方被使用。 

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

# POM 重构
#### dependencyManagement 维护依赖一致性
假如我们的开发模式是模块化开发，模块化开发即互相独立，低耦合性质。
如果每个模块都需要使用日志 log，可能每个模块的负责人引用的日志框架会不同。或者说假使使用了统一框架，但是版本不同。
此时会引起一个问题：难维护！

如何解决？

我们借助 maven 的 dependencyManagement，可以方便的解决这一问题。

两种实现方式：
- 子模块***继承***公共模块：`extends` 继承只能1对1

- 子模块***导入***公共模块：`import scope` 导入可以1对多

> 我们知道Maven的继承和Java的继承一样，是无法实现多重继承的。
如果10个、20个甚至更多模块继承自同一个模块，那么按照我们之前的做法，这个父模块的dependencyManagement会包含大量的依赖。
如果你想把这些依赖分类以更清晰的管理，那就不可能了，import scope依赖能解决这个问题。
你可以把dependencyManagement放到单独的专门用来管理依赖的POM中，然后在需要使用依赖的模块中通过import scope依赖，就可以引入dependencyManagement。

#### 子模块通过导入公共模块实现依赖管理


1.新建项目`common-module`，将公共依赖抽出去，统一管理（注意，packaging 的值必须为 pom）

```
<modelVersion>4.0.0</modelVersion>
<groupId>com.lfzhu</groupId>
<artifactId>common-module</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>pom</packaging>
    
<dependencyManagement>
        <!--junit-->
        <dependency>
            <groupId>org.hamcrest</groupId>
            <artifactId>hamcrest-all</artifactId>
            <version>1.3</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.mockito</groupId>
            <artifactId>mockito-core</artifactId>
            <version>1.9.5</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>

        <!--logback-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>1.7.7</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.1.2</version>
        </dependency>
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>4.8</version>
        </dependency>
    </dependencies>
</dependencyManagement>

```

这里，我们把日志框架，单元测试框架放在公共的 POM 中统一维护。

注意：***dependencyManagement 里面的配置不会给任何子模块引入依赖。***

打包并上传至公共仓库 nexus 中
```
$ mvn package
$ mvn deploy:deploy-file -DgroupId=com.lfzhu -DartifactId=common-module -Dversion=0.0.1-SNAPSHOT -Dpackaging=pom -Dfile=./pom.xml -DpomFile=./pom.xml -Durl=http://host:port/repository/maven-snapshots -DrepositoryId=nexus
```

2.新建项目`son-module-one`，子模块导入公共模块中的依赖

```
<modelVersion>4.0.0</modelVersion>
<groupId>com.lfzhu</groupId>
<artifactId>son-module-one</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>

<dependencyManagement>
    <dependencies>
         <!--可以导入多个项目-->
        <dependency>
            <groupId>com.lfzhu</groupId>
            <artifactId>common-module</artifactId>
            <version>0.0.1-SNAPSHOT</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
    
<dependencies>
    <!--junit-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
    </dependency>
</dependencies>             
```

当子模块需要使用JUnit的时候，我们就可以如此简化依赖配置。

只需要 groupId 和 artifactId，其它元素如 version 和 scope 都能通过导入的公共模块中的 dependencyManagement 得到。


#### 子模块通过继承公共模块实现插件(Plugin)管理
消除多模块插件配置重复

1.公共模块中添加配置

```
<build>
    <pluginManagement>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>2.3.2</version>
                <configuration>
                    <source>1.8</source>
                    <target>1.8</target>
                    <encoding>UTF-8</encoding>
                </configuration>
            </plugin>
        </plugins>
    </pluginManagement>
</build>
```

2.新建项目`son-module-two`，子模块中添加配置

```
<modelVersion>4.0.0</modelVersion>
<groupId>com.lfzhu</groupId>
<artifactId>son-module-two</artifactId>
<version>0.0.1-SNAPSHOT</version>
<packaging>jar</packaging>

<parent>
    <groupId>com.lfzhu</groupId>
    <artifactId>common-module</artifactId>
    <version>0.0.1-SNAPSHOT</version>
</parent>
```

由于 Maven 内置了 maven-compiler-plugin 与生命周期的绑定，因此子模块就不再需要任何 maven-compiler-plugin 的配置了。

注意：

- 简单的把插件配置提取到父 POM 的 pluginManagement 中往往不适合所有情况。

- 例如模块A运行所有单元测试，模块B要跳过一些测试，这时就需要配置 maven-surefire-plugin来实现，那样两个模块的插件配置就不一致了。此时不能使用 pluginManagement。

#### 参考
[Maven实战（三）——多模块项目的POM重构](http://www.infoq.com/cn/news/2011/01/xxb-maven-3-pom-refactoring)

# 使用 Maven 构建多模块(modules)项目
项目开发中，为了便于后期的维护，我们一般会进行分模块开发。各个模块之间的职责会比较明确，后期管理起来也相对比较容易。

#### 创建一个整体管理的项目 `system-parent`
创建完成之后，只保留POM文件

```
<modelVersion>4.0.0</modelVersion>
<groupId>com.lfzhu</groupId>
<artifactId>system-parent</artifactId>
<version>1.5-SNAPSHOT</version>
<packaging>pom</packaging>
<name>system-parent</name>
<url>http://maven.apache.org</url>

```

使用 `dependencyManagement` 管理依赖

```
<dependencyManagement>
    <dependencies>
        <!--logback-->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>jcl-over-slf4j</artifactId>
            <version>1.7.7</version>
        </dependency>
        <dependency>
            <groupId>ch.qos.logback</groupId>
            <artifactId>logback-classic</artifactId>
            <version>1.1.2</version>
        </dependency>
        <dependency>
            <groupId>net.logstash.logback</groupId>
            <artifactId>logstash-logback-encoder</artifactId>
            <version>4.8</version>
        </dependency>
    </dependencies>
</dependencyManagement>
```

#### 创建子模块A `system-module-a`
在 `system-parent` 项目上右击 `New Module`，选择 `maven` ，在 `artifactId`里面输入`system-module-a`

```
<modelVersion>4.0.0</modelVersion>
<artifactId>system-module-a</artifactId>
<version>1.4-SNAPSHOT</version>
<packaging>jar</packaging>
<name>system-domain</name>
<url>http://maven.apache.org</url>
```

此时，我们发现在POM文件里面多了 `parent` 标签
```
<parent>
    <artifactId>system-parent</artifactId>
    <groupId>com.lfzhu</groupId>
    <relativePath>../pom.xml</relativePath>
    <version>1.5-SNAPSHOT</version>
</parent>
```

快速引入依赖，使用父模块的依赖，无需输入版本号，自动使用父模块中该依赖的版本号
```
<dependencies>
    <!--junit-->
    <dependency>
        <groupId>junit</groupId>
        <artifactId>junit</artifactId>
    </dependency>
</dependencies>
```

#### 创建子模块B `system-module-b`
在 `system-parent` 项目上右击 `New Module`，选择 `maven` ，在 `artifactId`里面输入`system-module-b`

```
<modelVersion>4.0.0</modelVersion>
<artifactId>system-module-b</artifactId>
<version>1.3-SNAPSHOT</version>
<packaging>jar</packaging>
<name>system-service</name>
<url>http://maven.apache.org</url>

<parent>
    <artifactId>system-parent</artifactId>
    <groupId>com.lfzhu</groupId>
    <relativePath>../pom.xml</relativePath>
    <version>1.5-SNAPSHOT</version>
</parent>
```

如果模块B需要使用到模块A中的类，需要添加对模块A的依赖
```
<dependencies>
    <dependency>
        <groupId>com.lfzhu</groupId>
        <artifactId>system-module-a</artifactId>
        <version>1.4-SNAPSHOT</version>
    </dependency>
</dependencies>
``` 

#### 创建子模块C `system-module-c`
在 `system-parent` 项目上右击 `New Module`，选择 `maven` ，在 `artifactId`里面输入`system-module-c`

```
<modelVersion>4.0.0</modelVersion>
<artifactId>system-service</artifactId>
<!--不指定 version，默认取 system-parent 的 version 1.5-SNAPSHOT-->
<packaging>jar</packaging>
<name>system-service</name>
<url>http://maven.apache.org</url>

<parent>
    <artifactId>system-parent</artifactId>
    <groupId>com.lfzhu</groupId>
    <relativePath>../pom.xml</relativePath>
    <version>1.5-SNAPSHOT</version>
</parent>
```

如果模块C需要使用到模块A和模块B中的类，只需要添加对模块B的依赖，自动会将模块A添加进来，因为模块B已经依赖了模块A
```
<dependencies>
    <dependency>
        <groupId>com.lfzhu</groupId>
        <artifactId>system-module-b</artifactId>
        <version>1.3-SNAPSHOT</version>
    </dependency>
</dependencies>
``` 

#### 项目 `system-parent` 自动增加 `modules`标签
```
<modules>
    <module>system-module-a</module>
    <module>system-module-b</module>
    <module>system-module-c</module>
</modules>
```

#### 打包
###### 子模块独立打包
- 进入子模块的 `maven` 插件管理处，点击 `package` 

- 会在当前子模块项目中的 `target` 文件夹下生成 `jar` 文件

###### 父模块统一打包子模块
- 进入父模块的 `maven` 插件管理处，点击 `package` 

- 会在每个子模块项目中的 `target` 文件夹下生成各自对应的 `jar` 文件

- 父模块项目中不会生成 `target` 文件夹

#### 参考
[Maven学习总结(八)——使用Maven构建多模块项目](http://www.cnblogs.com/xdp-gacl/p/4242221.html)
