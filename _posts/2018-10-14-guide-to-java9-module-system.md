---
layout: post
title:  "Java9模块化实操指南&工具使用"
categories: Jigsaw
tags:  Jigsaw modularity
author: tiny
---

* content
{:toc}

## 前言

模块化是Java9正式引入的feature，全称是：Java Platform Module System (JPMS)，该特性的引入增强了Java的模块化和封装性；而且用户可利用JDK9提供的打包工具`Jlink`生成可执行镜像文件（该文件可不依JDK环境直接运行）。本文主要涉及到的内容如下：`module-info.java`的语法说明，`Jlink`, `Jmod`, `Jdeps`工具的使用；`Maven modules`与`Java modules`的关系；模块化实操练习DEMO；

## 语法说明

### JavaSE模块间依赖关系

![module graph](/images/posts/module-graph.png)

### 语法详解

- Java9模块化代码编写的核心类是`module-info.java`，该类必须在模块的根路径上定义（例如：maven项目中可以将`module-info.java`放置在`main\src\java`下）；

- 当前模块的所有定义都在`module-info.java`文件中，主要关键字包括：`exports`, `module`, `open`, `opens`, `provides`, `requires`, `uses` (`with`, `to`, `transitive`)；

- `module-info.java`中 模块名称必须定义且moduleName必须保证唯一（模块名可自定义，建议直接用模块包名做模块名），body（`{}`中的内容）可以为空；

- 当一个工程中有`module-info.java`时，会被当做一个模块来看待，访问外部模块中的类时会受外部模块定义的约束限制（例如：只能访问到其他模块`exports`的内容，更强的封装性），当工程中没有`module-info.java`时，则当成普通的jar访问；

- 默认情况下，在模块中的public类需要`exports`才能被外部其他模块访问到，`exports`的类中用public/protected修饰的嵌套类也可被外部模块访问到；

**各关键字使用说明如下：**

```java

/**
 * 声明模块名称：algorithm.api，open：可选项；
 * 当用open修饰module时表示模块中的任何类都可以被外部访问到；
 * all packages in a given module should be accessible at runtime and via
 * reflection to all other modules
 */
[open] module algorithm.api {   

  // 声明当前模块依赖于另一模块java.base
  // java.base是默认requires，不用在module-info.java中明确声明；
  requires java.base;

  // 依赖的传递性，与maven的依赖继承性相似，任何依赖algorithm.api的模块同时也依赖java.desktop;
  // 去掉transitive则必须在依赖模块中明确依赖，即声明requires java.desktop;
  // JavaSE内部module间的依赖都已用transitive修饰，这使得我们对JavaSE module声明依赖更加简洁;
  requires transitive java.desktop;

  // 对java.xml的依赖在编译期必须，运行期非必须，
  // 类似maven中的<scope>provided<scope>，使用Jlink打包的jimage不包含java.xml的文件
  requires static java.xml;

  // 将当前模块指定包中的public类（包含用public/protected修饰的嵌套类）exports（公布），供给外部模块访问；
  // 只exports当前声明的package中的类，子package中的内容不被导出，需另声明；
  exports nl.frisodobber.java9.jigsaw.calculator.algorithm.api;

  // 将包中的类导出给指定的Modules，只能在限定的Module内使用；
  exports nl.frisodobber.java9.jigsaw.calculator.algorithm.api.extension to java.desktop, java.sql, calculator.gui;

   // 引入通过 provides…with 提供的service类，一般 provides…with 的定义在其他模块中
   // 代码中可以通过 ServiceLoader.load(Algorithm.class); 获取可用的service类；
  uses nl.frisodobber.java9.jigsaw.calculator.algorithm.api.Algorithm;


   // 通过 provides…with 指令，提供一个实现类；外部模块使用时可用过 uses 引入；
  provides nl.frisodobber.java9.jigsaw.calculator.algorithm.api.Algorithm with nl.frisodobber.java9.jigsaw.calculator.algorithm.add.Add;

  // 所有包内的public类（包含public/protected嵌套类）只能在运行中被访问
  // 包内的所有类以及所有类内部成员都可以通过反射访问到
  opens nl.frisodobber.java9.jigsaw.calculator.algorithm.api;

  // 限定访问包的模块
  opens nl.frisodobber.java9.jigsaw.calculator.algorithm.api.scala to java.desktop, java.sql;

  /**
   * By default, a module with runtime reflective access to a package can see the package’s public types
   * (and their nested public and protected types). However, code in other modules can access all types in
   * the exposed package and all members within those types, including private members via setAccessible,
   * as in earlier Java versions.
   */

}

```

## 工具使用
### Java & Javac

可以通过`java -h`, `javac --help`查看所有与模块化相关的命令；

- `java --list-modules`：列出可见模块；
- `java -d <moduleName>` or `java --describe-module <moduleName>`：描述模块，`moddule-info.java`中定义的信息；  

- 打包一个模块  

  - 手动打包模块：

  ```shell
  # 首先编译源码  
  > cd maven-java9-jigsaw
  > javac
    --module-path fd-java9-jigsaw-algorithm-api/src/main/java
    -d classes/api
    fd-java9-jigsaw-algorithm-api/src/main/java/nl/frisodobber/java9/jigsaw/calculator/algorithm/api/Algorithm.java
  ```

  ```shell
  # 其次打包jar文件
  # TODO 需将下述cmd替换上一步打编译的内容
  > jar --create
      --file target/jpms-hello-world.jar
      --main-class org.codefx.demo.jpms.HelloModularWorld
      -C target/classes      
  ```
  - maven工程直接用maven打包命令：`mvn clean package`  

  **注：自己手动打包方式比较麻烦，实际项目中直接用 maven or gradle.**  

- 执行一个模块
`java --module-path <path1>[;<path2>...] -m <moduleName>/<mainClass>`  
`--module-path`: 执行模块所处的路径，多个路径用 `;` 分割；  
`-m` or `--module`：指定要执行的模块 & 要执行的main函数所在的类；

### Jlink
- 用`Jlink`创建一种名为`jimage`的镜像文件，可直接运行，无需JDK环境；`jimage`可以有效减小运行时镜像（剔除了无依赖的`java module`）；
- `Jlink`使用时要求当前项目以及其所依赖的所有jar都有`module-info.java`文件（针对无`module-inf.java`的jar存在的场景解决的解决方案可关注视频靠后一部分的讲解： [https://youtu.be/jpi2i1d7hqc](https://youtu.be/jpi2i1d7hqc)）；

生成`jimage`示例：
```shell
> cd maven-java9-jigsaw
> mvn clean package -DskipTests
> jlink
  --module-path libs
  --add-modules calculator.gui,calculator.cli
  --compress 2  // gzip压缩，可选
  --output jimage
```

运行`jimage`：
```shell
> cd jimage/bin
> java -m calculator.gui/nl.frisodobber.java9.jigsaw.calculator.gui.Main
```

### Jmod

// TODO diff between jmod and jar, how to use jmod;

生成jmod文件：
```shell
# 将jar转换成jmod文件
> cd libs
> jmod
  create calculator.gui
  --class-path fd-java9-jigsaw-gui-1.0-SNAPSHOT.jar
  --module-version 1.0
```
### Jdeps
依赖对象分析工具，可分析模块相关依赖信息，具体命令可通过`jdeps -h`查看，简单示例：

```shell
> cd libs
> jdeps
  --module-path libs
  -m calculator.gui
  --list-deps
```

### Maven Plugins

| name     | description     |
| :------------- | :------------- |
| [jlink](http://maven.apache.org/plugins/maven-jlink-plugin/)   | Build Java Run Time Image |
| [jmod](http://maven.apache.org/plugins/maven-jmod-plugin/)   | Build Java JMod files |

## Maven Vs Java modules
- maven的功能是打包&依赖管理（包含依赖包的存放/版本管理等）；
- `java9 module system`的功能是模块的管理（定义&继承&类访问合法性约束）；
- maven中的模块一般指代一个完整功能体，比如一个jar包&jar所有依赖；`java9 module`则特指`module-info.java`中定义的模块；
- 针对`java module`的工程来讲，maven仍是作为jar包管理&打包的工具，但java模块间的依赖管理&类访问约束是依赖`java module system`管理；
- maven中有提供`plugin`用于方便的编译&打包`java module`，但是`alpha`版本，尚未release `GA`；
- 一个`JPMS` vs `maven`的传送门：
[https://stackoverflow.com/questions/39844602/project-jigsaw-vs-maven](https://stackoverflow.com/questions/39844602/project-jigsaw-vs-maven)


## 模块化DEMO练习
可参照工程：[https://github.com/itinycheng/maven-java9-jigsaw](https://github.com/itinycheng/maven-java9-jigsaw)


## 总结
本章主要目的是讲解模块化的入门实操和模块化相关的工具使用，经过本章的学习相信大家可以顺利的在自己项目中引入模块化开发。为更方便直观的学习，我Fork了一个比较好的模块化Demo，并修改了原项目中的错误 & 升级Java版本到11，大家可以在该项目基础上做些实操练习，巩固所学，项目地址： https://github.com/itinycheng/maven-java9-jigsaw。

**addition:**  
`JPMS`特性引入之后，`Java`在运行时也能获取`Module`相关信息，具体可关注`Class.java`中新增的变量/方法，比如：`clazz.getModule()`,`clazz.forName(module, name)`, etc.


## 参考

- https://openjdk.java.net/projects/jigsaw/
- https://www.oracle.com/corporate/features/understanding-java-9-modules.html
- https://dzone.com/articles/java-9-modules-introduction-part-1
- https://dzone.com/articles/java-9-modules-part-2-intellij-and-maven
- https://dzone.com/articles/java-9-modules-part-3-directives
- https://dzone.com/articles/jlink-in-java-9
- https://dzone.com/articles/getting-started-with-java-9-modulesproject-jigsaw
- https://stackoverflow.com/questions/39844602/project-jigsaw-vs-maven
