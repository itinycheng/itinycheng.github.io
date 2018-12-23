---
layout: post
title:  "Scala类型参数详解&对比Java泛型体系"
categories: generic
tags:  generic scala java
author: tiny
---

* content
{:toc}

## 前言

`Scala`类型参数与`Java`的泛型体系十分相似，但比`Scala`类型参数体系的实现比后者更为的多样和复杂；Scala类型参数体系包含如下：[多重]上界（`T <: UpperBound`）、[多重]下界（`T >: LowerBound`）、[多重]视图界定（`T <% ViewBound`）、[多重]上线文界定（`T : ContextBound`）、类型约束（`T =:= U`, `T <:< U`, `T <%< U`）、协变（`+T`）、逆变（`-T`）；可以认为`Scala`类型参数在覆盖Java泛型所有特性的基础上又做了更多的扩展延伸，即`Java`泛型是`Scala`类型参数的一个子集。本文主要讲解`Scala`参数类型的各种特性并辅助与`Java`泛型做对比分析。

## 语法讲解

**首先定义若干类结构**

```scala
trait Food
trait Medicine
class Fruit extends Food
class Apple extends Fruit
// 橘子是一种水果，橘子皮有药用价值
class Orange extends Fruit with Medicine
```

### [多重]上/下界（upper/lower bound）

`T <: upperBound` - 类型上界
`T >: lowerBound` - 类型下界
`T >: lowerBound <: upperBound` - 多重界定
`with` - 连接多个上界或多个下界   

1. 用符号`<:`来表示类型参数的上界，如`_ <: Food` 表示任意类型`_`是Food的子类，默认情况下`Scala`类的上界是`Any`（类似`Java`中的`Object`），即任意类型参数可表示为`_ <: Any`。
`scala`支持同时声明多个上界，两个上界类型之前用`with`连接，例如：`_ <: Fruit with Medicine`给任意类型`_`定义了两个上界：`Fruit`, `Medicine`，在上文定义的类中只有`Orange`满足这个多重上界的限定。`with`可以多次使用，用以连接多个上界类型，语法如下：`_ <: TypeA with TypeB with TypeC with ...`。  

2. 用符号`>:`来表示类型参数的下界，如：`_>:Fruit`表示任意类型`_`需满足是`Fruit`的父类的上界限定，`scala`也支持多重上界的语法，如：`_ >: Fruit with Food`表示任意类型`_`必须同时满足是`Fruit`和`Food`的父类，上文定义类中无满足该要求的类，`AnyRef`可以满足这个多重上界的要求（`scala`中`AnyRef`是任意引用类型的父类）。

3. 上界和下界一起使用可被称为多重界定，即一个类型参数时有上界和下界，如：`_ >: Apple <: Food`表示任意类型`_`需满足以`Apple`作为下界，同时以`Food`作为上界，上文定义的类中`Food`满足该限定；此外，一个类型参数也可以同时拥有多上界和多个下界，如：`_ >: Apple with Food <: AnyRef with Serializable`定义了两个下界`Apple`和`Food`，以及两个上界`AnyRef`和`Serializable`。

- 代码示例

```scala

/**
  * 函数接收一个ListBuffer类型的入参，ListBuffer存放的对象必须是Fruit子类
  */
def test0(list: ListBuffer[_ <: Fruit]): Unit = {
  // compile error: list ++= List(new Fruit,new Orange,new Apple)
  list.foreach(println)
}

/**
  * define a generic type `T` which has an upper bounds `Fruit`
  */
def test1[T <: Fruit](list: ListBuffer[T]): Unit = {
  // compile error :list ++= List(new Fruit,new Orange,new Apple)
  list.foreach(println)
}

/**
 * 函数接收一个ListBuffer类型入参，ListBuffer存放对象必须是Fruit父类
 */
def test2(list: ListBuffer[_ >: Fruit]): Unit = {
    list ++= List(new Apple, new Fruit)
    list.foreach(println)
}

/**
  * @see [[test2]]
  */
def test3[T >: Fruit](list: ListBuffer[T]): Unit = {
  list ++= List(new Apple, new Fruit)
  list.foreach(println)
}

/**
  * multi upper bounds
  */
def test4[T <: Fruit with Medicine](list: ListBuffer[T]): Unit = {
    //compile error: list ++= List(new Orange)
    list.foreach(println)
}

/**
  * multi lower bounds
  */
def test5[T >: Fruit with Medicine](list: ListBuffer[T]): Unit = {
    list ++= List(new Orange)
    list.foreach(println)
}

/**
  * have upper & lower bounds at the same time
  */
def test6[T >: Apple <: Food](list: ListBuffer[T]): Unit = {
  // compile error: list += new Fruit
  list.foreach(println)
}

/**
  * multi upper & lower bounds
  */
def test7[T >: Apple with Food <: AutoCloseable with Serializable](list: ListBuffer[T]): Unit = {
  // compile error: list ++= List(new Orange, new Apple)
  list.foreach(println)
}

```

**实际上：** 上界对应的是`Java`中的`extends`关键字，下界对应`Java`中的`super`关键字，`scala`的上下界语法特性比`Java`的语法表现更全面；在`Java`中无法定义一个泛型`<T super Fruit>`（可能在`Java`看来`super of every class is Object`）；但`Java`支持多重的上界，比如定义一个泛型`T`必须继承两个类`<T extends Fruit & Medicine>`，其中`&`与`scala`中的`with`关键字相对。将`scala`源码编译成字节码，然后反编译字节码进行观察发现之前定义的带有上下界的类型参数只剩下`extends`关键字（对应`<:`），而`super`关键字（对应`>:`）基本被擦除（`Be Erased`），从这一层面来讲，上下界是`scala`特有的语法层特性，是编译时特性非运行时，`scala`源码编译成字节码时会进行语法表达的转换和裁切，以符合JVM的字节码规范。

### [多重]视图界定（view bound）

`T <% viewBound` - 视图界定（deprecated from scala 2.11）

**首先定义若干类和隐私转换**

```scala

class Juice(food: Fruit){
  def juice = "Juice"
}
class Soup(food: Fruit){
  def soup = "Soup"
}

/**
  * 水果可以榨汁
  */
implicit def fruitToJuice(fruit: Fruit): Juice = {
  new Juice(fruit)
}
/**
  * 水果可以做汤
  */
implicit def fruitToSoup(fruit: Fruit): Soup = {
  new Soup(fruit)
}

```

1. 用符号`<%`来表示视图界定，`T <% Juice`表示在当前`.scala`文件的上下文中，存在一个隐式函数可以将类型`T`转换为`Juice`。

2. `scala`语法也支持多重的视图界定，如：`T <% Juice <% Soup`表示类型`T`既可以隐式转换成`Juice`也可以转换成`Soup`。

**注：** 视图界定在`Scala 2.11`版本已经`deprecated`，请用隐式参数替换，如下代码示例中的`test2`, `test3`；

- 代码示例

```scala

// 引入隐式转换函数
import fruitToJuice, fruitToSoup

/**
  * 该函数要求传入的ListBuffer中的T的对象可以隐式转换成Juice
  * view bounds are deprecated,use implicit parameter instead.
  * input parameter `T` can convert to `Juice`
  */
def test0[T <% Juice](list: ListBuffer[T]): Unit = list.foreach(i => println(i.juice))

/**
  * multi view bound
  */
def test1[T <% Juice <% Soup](list: ListBuffer[T]): Unit = {
  // 通过隐式转换成Juice而拥有juice函数
  list.foreach(i => println(i.juice))
  // 通过隐式转换成Juice而拥有soup函数
  list.foreach(i => println(i.soup))
}

/**
  * Use this function to replace [[test0()]]
  */
def test2[T](list: ListBuffer[T])(implicit fun: T => Juice): Unit = list.foreach(i => println(i.juice))


/**
  * Use this function to replace [[test1()]]
  */
def test3[T](list: ListBuffer[T])(implicit f1: T => Juice, f2: T => Soup): Unit = {
  list.foreach(i => println(i.juice))
  list.foreach(i => println(i.soup))
}

// 调用函数
def main(args: Array[String]): Unit = {
    val fruits = ListBuffer[Fruit](new Apple, new Orange, new Fruit)
    test0(fruits)
    test1(fruits)
    test2(fruits)
    test3(fruits)
}

```

**实际上：** 视图界定是`scala`为了让coding更简洁高效而设计出的一个语法糖，其实现是`scala`编译器在编译生成字节码过程中的一个自动插入隐式函数到所需位置的操作；如在上边的代码中`ListBuffer[Fruit]`作为入参传入`test0`，在`foreach`时`Fruit`对象是没有`juice`函数供给调用的，编译这段代码时不通过，这时编译器会从上下文中找隐式转换，找到了`fruitToJuice`，并调用该函数将`fruit`对象转换为`Juice`，然后在调用`Juice`中的`juice`函数（.class反编译结果类似`list.foreach(i => println(fruitToJuice(i).juice))`）；`Java`中无对应特性与之对应，即每个调用必须明确，编译器并不做类似自动化的补全操作。

### 上下文界定（context bound）

`T: contextBound` - 上下文界定（存在`ContextBound[T]`的隐式值）

**首先定义隐式对象**

```scala

implicit object FruitOrdering extends Ordering[Fruit] {
  override def compare(x: Fruit, y: Fruit): Int = {
    x.getClass.getName.compareTo(y.getClass.getName)
  }
}

```

1. 上下文界定的形式为`T : M`，其中`M`是另一个泛型类，它要求必须存在一个类型为`M[T]`的隐式值。如：`[T : Ordering]`表示存在一个`Ordering[Fruit]`的隐式值，

define 隐式值 ？？

### 类型约束

### 不变（invariance）

### 协变（covariant）

### 逆变（contravariance）

## 总结

- compare

| Header One     | Header Two     |
| :------------- | :------------- |
| Item One       | Item Two       |

## 参考
