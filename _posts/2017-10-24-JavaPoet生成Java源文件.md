---
layout:     post
title:      JavaPoet 生成 Java 源文件
subtitle:   使用 JavaPoet 框架快速生成 Java 源文件
date:       2017-10-24
author:     uncledrew
header-img: img/post-bg-re-vs-ng2.jpg
catalog: true
tags:
    - Spring
    - JavaPoet
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrewzhu.github.io/)

# 先看两个例子

#### 生成一个枚举类

代码：
```
import com.squareup.javapoet.ClassName;
import com.squareup.javapoet.FieldSpec;
import com.squareup.javapoet.JavaFile;
import com.squareup.javapoet.MethodSpec;
import com.squareup.javapoet.TypeName;
import com.squareup.javapoet.TypeSpec;

import javax.lang.model.element.Modifier;
import java.io.File;
import java.io.IOException;

public class EnumGenerator {

    private void generatorEnum() {
        TypeName enumDemo = ClassName.get("com.lfzhu.quickstart.poet", "Fun21");
        TypeSpec coffee = TypeSpec.enumBuilder("Coffee")
            .addModifiers(Modifier.PUBLIC,Modifier.STATIC)
            .addSuperinterface(enumDemo)
            .addEnumConstant("BLACK_COFFEE")
            .addEnumConstant("DECAF_COFFEE")
            .addEnumConstant("LATTE")
            .build();

        TypeSpec light = TypeSpec.enumBuilder("Light")
            .addModifiers(Modifier.PUBLIC,Modifier.STATIC)
            .addSuperinterface(enumDemo)
            .addEnumConstant("RED",TypeSpec.anonymousClassBuilder("$S", "RED").build())
            .addEnumConstant("YELLOW",TypeSpec.anonymousClassBuilder("$S", "YELLOW").build())
            .addField(FieldSpec.builder(String.class, "value")
                .addModifiers(Modifier.PRIVATE).build())
            .addMethod(MethodSpec.constructorBuilder().addModifiers(Modifier.PRIVATE)
                .addParameter(String.class, "value").build())
            .addMethod(MethodSpec.methodBuilder("toString").addModifiers(Modifier.PUBLIC)
                .addAnnotation(Override.class).returns(String.class)
                .addStatement("return this.value").build())
            .build();

        TypeSpec typeSpec = TypeSpec.interfaceBuilder("EnumDemo")
            .addModifiers(Modifier.PUBLIC)
            .addType(coffee)
            .addType(light)
            .build();

        JavaFile javaFile = JavaFile.builder("com.lfzhu.quickstart.poet", typeSpec)
            .build();
        try {
            File file = new File("src/main/java");
            javaFile.writeTo(file);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        EnumGenerator enumGenerator = new EnumGenerator();
        enumGenerator.generatorEnum();
    }

}
```

运行结果：
```
import java.lang.Override;
import java.lang.String;

public interface EnumDemo {
  enum Coffee implements Fun21 {
    BLACK_COFFEE,

    DECAF_COFFEE,

    LATTE
  }

  enum Light implements Fun21 {
    RED("RED"),

    YELLOW("YELLOW");

    private String value;

    private Light(String value) {
    }

    @Override
    public String toString() {
      return this.value;
    }
  }
}
```

#### 生成一个 Spring 控制器类

代码：
```
import com.squareup.javapoet.AnnotationSpec;
import com.squareup.javapoet.ClassName;
import com.squareup.javapoet.FieldSpec;
import com.squareup.javapoet.JavaFile;
import com.squareup.javapoet.MethodSpec;
import com.squareup.javapoet.ParameterSpec;
import com.squareup.javapoet.TypeSpec;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

import javax.lang.model.element.Modifier;
import java.io.File;
import java.io.IOException;

public class ControllerGenerator {

    private void generatorController() {
        ParameterSpec userId = ParameterSpec.builder(Long.class, "userId")
            .addAnnotation(RequestParam.class)
            .build();

        ClassName className = ClassName.get("com.lfzhu.quickstart.poet.staticclass", "MyUser");
        MethodSpec methodSpec = MethodSpec.methodBuilder("getUserInfo")
            .addModifiers(Modifier.PUBLIC)
            .returns(className)
            .addAnnotation(AnnotationSpec.builder(RequestMapping.class)
                .addMember("value", "$S", "/getUserInfo")
                .addMember("method", "$T.GET", RequestMethod.class)
                .build())
            .addAnnotation(ResponseBody.class)
            .addParameter(userId)
            .addStatement("$T user = new $T()", className, className)
            .addStatement("user.setId($N)", "userId")
            .addStatement("user.setName($S + $N)", "tom", "userId")
            .addStatement("return user")
            .build();

        ClassName userServiceClass = ClassName.get("com.lfzhu.quickstart.poet.staticclass", "UserService");
        FieldSpec fun16 = FieldSpec.builder(userServiceClass, "userService")
            .addModifiers(Modifier.PRIVATE)
            .addAnnotation(Autowired.class)
            .build();

        TypeSpec typeSpec = TypeSpec.classBuilder("ControllerDemo")
            .addModifiers(Modifier.PUBLIC)
            .addAnnotation(Controller.class)
            .addAnnotation(AnnotationSpec.builder(RequestMapping.class)
                .addMember("value", "$S", "/UserManage")
                .build())
            .addField(fun16)
            .addMethod(methodSpec)
            .build();

        JavaFile javaFile = JavaFile.builder("com.lfzhu.quickstart.poet", typeSpec)
            .build();
        try {
            File file = new File("src/main/java");
            javaFile.writeTo(file);
        } catch (IOException e) {
            e.printStackTrace();
        }
    }

    public static void main(String[] args) {
        ControllerGenerator controllerGenerator = new ControllerGenerator();
        controllerGenerator.generatorController();
    }

}
```

运行结果：
```
import com.lfzhu.quickstart.poet.staticclass.MyUser;
import com.lfzhu.quickstart.poet.staticclass.UserService;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.ResponseBody;

@Controller
@RequestMapping("/app/api/UserManage")
public class ControllerDemo {
  @Autowired
  private UserService userService;

  @RequestMapping(
      value = "/getUserInfo",
      method = RequestMethod.GET
  )
  @ResponseBody
  public MyUser getUserInfo(@RequestParam Long userId) {
    MyUser user = new MyUser();
    user.setId(userId);
    user.setName("tom" + userId);
    return user;
  }
}
```

# JavaPoet 是什么
原作者是这么介绍 JavaPoet 的。`JavaPoet is a Java API for generating .java source files.`

翻译过来就是 `JavaPoet 是生成 Java 源文件的一个API.`

# JavaPoet 如何使用
JavaPoet 框架中最基础的 API 接口，都可以参考 GitHab 上的 [square/javapoet](https://github.com/square/javapoet)这篇文章。
这里面有详细的介绍和教程，基本上覆盖了一个 Java 源文件中所涉及的所有语法语句。

四个常用标签：
- $L：替换一串文字  `("int i = $L", 1)` -> `(int i = 1;)`
- $S：替换一个字符串  `("String s = $S", "drew")` -> `(String s = "drew";)`
- $T：替换一个类型  `("return new $T()", Date.class)` -> `(return new Date();)`
- $N：替换一个其名称生成的另一个声明  `MethodSpec fun = ....; (String result = $N()", fun)` -> `(String result = fun();)`

四个常用类：
- MethodSpec：声明一个构造函数或方法
- TypeSpec：声明一个类、接口或者枚举
- FieldSpec：声明一个成员变量或者一个字段
- JavaFile：声明一个顶级类的 Java 文件

# Maven 项目如何引入 JavaPoet
```
<dependency>
    <groupId>com.squareup</groupId>
    <artifactId>javapoet</artifactId>
    <version>1.7.0</version>
</dependency>
```

# JavaPoet 有何优势
- 动态生成各种复杂的 Java 源文件
- 提高开发效率，减少重复的工作
- 规范程序的代码格式
- 规范项目的开发流程


