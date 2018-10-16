---
layout: post
title:  "java9模块化系统入门实操"
categories: Jigsaw
tags:  Jigsaw modularity
ser_start: jekyll serve
author: tiny
---

* content
{:toc}

## 前言

模块化是Java9正式引入的feature，全称是：Java Platform Module System (JPMS)，该特性的引入增强了Java的模块化和封装性；而且用户可利用JDK9提供的打包工具`Jlink`生成可执行镜像文件（该文件可不依JDK环境直接运行）。本文主要涉及到的内容如下：`module-info.java`的语法说明，`Jlink`, `Jmod`, `Jdeps`工具的使用；`Maven modules`与`Java modules`的关系；模块化实操练习DEMO；

## 语法说明

### JavaSE模块间依赖关系

![module graph](../images/posts/module-graph.png)

### 语法说明

- Java9模块化代码编写的核心类是`module-info.java`，该类必须在模块的根路径上定义（例如：maven项目中可以将`module-info.java`放置在`main\src\java`下）；

- 当前模块的所有定义都在`module-info.java`文件中，主要关键字包括：`exports`, `module`, `open`, `opens`, `provides`, `requires`, `uses` (`with`, `to`, `transitive`)；

- `module-info.java`中 模块名称必须定义且moduleName必须保证唯一（模块名可自定义，建议直接用模块包名做模块名），body（`{}`中的内容）可以为空；

- 当一个工程中有`module-info.java`时，会被当做一个模块来看待，访问外部模块中的类时会受外部模块定义的约束限制（例如：只能访问到其他模块`exports`的内容，更强的封装性），当工程中没有`module-info.java`时，则当成普通的jar访问；

- 默认情况下，在模块中的public类需要`exports`才能被外部其他模块访问到，`exports`的类中用public/protected修饰的嵌套类也可被外部模块访问到；

```java

//
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
  // TODO 类似maven中的<scope>provided<scope> ? 使用Jlink插件打包时的效果是？
  requires static java.xml;

  // 将当前模块指定包中的public类（包含用public/protected修饰的嵌套类）exports（公布），供给外部模块访问；
  // 只exports当前声明的package中的类，子package中的内容不被导出，需另声明；
  exports nl.frisodobber.java9.jigsaw.calculator.algorithm.api;

  // 将包中的类导出给指定的Modules，只能在限定的Module内使用；
  exports nl.frisodobber.java9.jigsaw.calculator.algorithm.api.scala to java.desktop, java.sql, calculator.gui;

  /**
   * TODO
   * A uses module directive specifies a service used by this module—making the module a service consumer.
   * A service is an object of a class that implements the interface or extends the abstract class specified in
   * the uses directive
   */
  uses Algorithm;

  /** TODO
   * provides…with. A provides…with module directive specifies that a module provides a service
   * implementation—making the module a service provider. The provides part of the directive specifies an interface
   * or abstract class listed in a module’s uses directive and the with part of the directive specifies the name of
   * the service provider class that implements the interface or extends the abstract class.
   */
  provides Algorithm with Impl;

  // 所有包内的public类（包含public/protected嵌套类）只能在运行中被访问
  // 包内的所有类以及所有类内部成员都可以通过反射访问到
  opens nl.frisodobber.java9.jigsaw.calculator.algorithm.api;
  // 限定访问包的模块
  opens nl.frisodobber.java9.jigsaw.calculator.algorithm.api.scala to java.desktop, java.sql;

  /**
   * TODO
   * By default, a module with runtime reflective access to a package can see the package’s public types
   * (and their nested public and protected types). However, the code in other modules can access all types in
   * the exposed package and all members within those types, including private members via setAccessible,
   * as in earlier Java versions.
   */

}

```

###
## 工具使用

### Java --module-path
### Jlink
### Jdeps
### Jmod
### Maven Plugins


## Maven Vs Java modules

## 工程模块化示例


## 总结

## 参考

- https://openjdk.java.net/projects/jigsaw/
- https://www.oracle.com/corporate/features/understanding-java-9-modules.html
- https://dzone.com/articles/java-9-modules-introduction-part-1
- https://dzone.com/articles/java-9-modules-part-2-intellij-and-maven
- https://dzone.com/articles/java-9-modules-part-3-directives
- https://dzone.com/articles/jlink-in-java-9
- https://dzone.com/articles/getting-started-with-java-9-modulesproject-jigsaw
- https://stackoverflow.com/questions/39844602/project-jigsaw-vs-maven
