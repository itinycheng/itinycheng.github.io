---
layout: post
title:  "系统讲解Scala&Java的协变与逆变"
categories: generic
tags:  generic scala covariant contravariance
author: tiny
---

* content
{:toc}

## 前言
第一次听说`协变&逆变`是刚接触`Scala`的时候，但`协变&逆变`却不是`Scala`所特有的，`Java`, `C#`等语言也有`协变&逆变`的概念，本文首先会解释`协变&逆变`这两个术语的含义，然后进一步探讨它们在`Java`, `Scala`中具体的书写语法和使用场景。

## 术语讲解
`协变&逆变`的概念入门可以关注下边这两个篇文章，本段内容也是学习这两篇文章所得到的：[Covariance and Contravariance of Hosts and Visitors](https://www.clear.rice.edu/comp310/JavaResources/generics/co_contra_host_visitor.html), [Cheat Codes for Contravariance and Covariance](https://www.originate.com/thinking/stories/cheat-codes-for-contravariance-and-covariance/)。

- 定义三个`实体`：`Fuel`,` Plant`, `Bamboo`；
- 定义`Box`：用来存放`Fuel`,` Plant`, `Bamboo`；
![entities](/images/posts/variance/entities.png)
- 定义三个`消费者`可消费实体的类型`Burn`, `Animal`, `Panda`；
![consumers](/images/posts/variance/consumers.png)
<br/>

- **协变**
`Burn`可以燃烧一切包括：`Fuel`,` Plant`, `Bamboo`；
`Animal`可以吃所有类型的植物包括：`Plant`, `Bamboo`；
`Panda`只吃一种植物：`Bamboo`；
我们可以将装有不同实体的`Box`传递给不同消费者，由此可以得出下面这张图：
![covariance-of-boxes](/images/posts/variance/covariance-of-boxes.png)

```java

  class BoxVisitor<U, R>{
    public R visit(Box<? extends U> host){
    }
  }

  class Panda<R> extends BoxVisitor<Bamboo, R>{
  }

```
- **逆变**
可以将装满`Fuel`的`Box`，传递给`Burn`（汽油只可以燃烧）；
可以将装满`Plant`的`Box`传递给`Burn`, `Animal`（植物可以燃烧&吃）；
可以将装满`Bamboo`的`Box`传递给`Burn`, `Animal`, `Panda`（竹子可以燃烧，可以给动物&熊猫吃）；
由此我们可以得出下面的这张图：
![covariance-of-boxes](/images/posts/variance/contravariance-of-visitors.png)

## 语法讲解

### 协变（covariant）

### 不变（invariance）

### 逆变（contravariance）
