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
    - Annotation
    - 注解
    - 枚举
---

> 纸上得来终觉浅，绝知此事要躬行。
>
> [我的博客](http://uncledrewzhu.github.io/)

# 先看两个例子

#### 生成一个枚举

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

#### 生成一个控制器

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
                .addMember("value", "$S", "/app/api/UserManage")
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
