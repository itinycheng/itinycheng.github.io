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

`Scala`类型参数与`Java`的泛型体系是较相似的语言特性，但`Scala`类型参数体系的实现比`Java`更为的多样和复杂；`Scala`类型参数体系包含如下：[多重]上界（`T <: UpperBound`）、[多重]下界（`T >: LowerBound`）、[多重]视图界定（`T <% ViewBound`）、[多重]上下文界定（`T : ContextBound`）、类型约束（`T =:= U`, `T <:< U`, `T <%< U`）、协变（`+T`）、逆变（`-T`）；可以认为`Scala`类型参数在覆盖Java泛型所有特性的基础上又做了更多的扩展延伸，即`Java`泛型是`Scala`类型参数的一个子集。本文主要讲解`Scala`参数类型的各种特性并辅助与`Java`泛型做对比分析，是一个供给Java Developer学习用的`Scala`类型参数入门指南。

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

- `T <: upperBound` 类型上界
- `T >: lowerBound` 类型下界
- `T >: lowerBound <: upperBound` 多重界定
- `with` 连接多个上界或多个下界   

1. 用符号`<:`来表示类型参数的上界，如`_ <: Food` 表示任意类型`_`是`Food`的子类，默认情况下`Scala`类的上界是`Any`（类似`Java`中的`Object`），即任意类型参数可表示为`_ <: Any`。
`scala`支持同时声明多个上界，两个上界类型之前用`with`连接，例如：`_ <: Fruit with Medicine`给任意类型`_`定义了两个上界：`Fruit`, `Medicine`，在上文定义的类中只有`Orange`满足这个多重上界的限定。`with`可以多次使用，用以连接多个上界类型，语法如下：`_ <: TypeA with TypeB with TypeC with ...`。  

2. 用符号`>:`来表示类型参数的下界，如：`_>:Fruit`表示任意类型`_`需满足是`Fruit`的父类的上界限定，`scala`也支持多重上界的语法，如：`_ >: Fruit with Food`表示任意类型`_`必须同时满足是`Fruit`和`Food`的父类，上文定义类中无满足该要求的类，`AnyRef`可以满足这个多重上界的要求（`scala`中`AnyRef`是任意引用类型的父类）。

3. 上界和下界一起使用可被称为多重界定，即一个类型参数时有上界和下界，如：`_ >: Apple <: Food`表示任意类型`_`需满足以`Apple`作为下界，同时以`Food`作为上界，上文定义的类中`Food`满足该限定；此外，一个类型参数也可以同时拥有多上界和多个下界，如：`_ >: Apple with Food <: AnyRef with Serializable`定义了两个下界`Apple`和`Food`，以及两个上界`AnyRef`和`Serializable`。

- **代码示例**

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

- `T <% viewBound` 视图界定（deprecated from `scala 2.11`）

1. 用符号`<%`来表示视图界定，`T <% Juice`表示在当前`.scala`文件的上下文中，存在一个隐式函数可以将类型`T`转换为`Juice`。

2. `scala`语法也支持多重的视图界定，如：`T <% Juice <% Soup`表示类型`T`既可以隐式转换成`Juice`也可以转换成`Soup`。

**注：** 视图界定在`Scala 2.11`版本已经`deprecated`，请用隐式参数替换，如下代码示例中的`test2`, `test3`；

- **首先定义若干类和隐私转换**

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

- **代码示例**

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

### [多重]上下文界定（context bound）

- `T: contextBound` 上下文界定（存在`ContextBound[T]`的隐式值）

1. 上下文界定的形式为`T : M`，其中`M`是另一个泛型类，它要求必须存在一个类型为`M[T]`的隐式值。如：`[T : Ordering]`表示必须存在一个`Ordering[Fruit]`的隐式值，在使用隐式值的地方声明`隐式参数`（如：`(implicit ordering: Ordering[T])`）。  

2. 多重上下文界定形式为`T : M : N`，表示当前代码上下文中必须存在类型`M[T]`和`N[T]`两个隐式值，同样，在使用隐式值的地方可以显式的声明`隐式参数`。

**首先定义隐式值**

```scala

implicit object FruitOrdering extends Ordering[Fruit] {
  override def compare(x: Fruit, y: Fruit): Int = {
    x.getClass.getName.compareTo(y.getClass.getName)
  }
}

/**
 * alternative
 *
 * has the same effect as FruitOrdering， but not singleton， better use `FruitOrdering`
 */
implicit def ordering: Ordering[Fruit] = new Ordering[Fruit] {
  override def compare(x: Fruit, y: Fruit): Int = x.getClass.getName.compareTo(y.getClass.getName)
}

```

- **代码示例**

```scala

import FruitOrdering
import scala.reflect.ClassTag

def test0[T: Ordering](first: T, second: T): Unit = {
  // compile error: val arr = new Array[T](2)
  println(first + ", " + second)
}

/**
  * use `implicit` parameter or else `Fruit extends Ordering[Fruit]`
  * 必须存在一个类型为`Ordering[T]`的隐式值
  */
def test1[T: Ordering](first: T, second: T)(implicit ordering: Ordering[T]): Unit = {
  // compile error: val arr = new Array[T](2)
  val small = if (ordering.compare(first, second) < 0) first else second
  println(small)
}

/**
 * ClassTag 用于保存运行时`T`的实际类型,`new Array[T](2)`就可以正常编译通过
 *
 * scala command compile result：
 * `test2: [T](first: T, second: T)(implicit evidence$1: Ordering[T], implicit evidence$2: scala.reflect.ClassTag[T])Unit`
 */
def test2[T: Ordering : ClassTag](first: T, second: T)(implicit ordering: Ordering[T]): Unit = {
  val arr = new Array[T](2)
  val small = if (ordering.compare(first, second) < 0) first else second
  println(arr + ", " + small)
}

// 调用函数
def main(args: Array[String]): Unit = {
    test0(new Apple, new Orange)
    test1(new Apple, new Orange)
    test2(new Apple, new Orange)
}

```

**说明：**

- 关于隐式转换，隐式参数相关内容本文并没做太多的讲解，网上相关资料比较多，大家可以尝试自助学习下；

- 将上述函数`test0`输入到`scala command`得到的结果为`test0: [T](first: T, second: T)(implicit evidence$1: Ordering[T])Unit`，即`[:Ordering]`被编译成了隐式参数：`implicit evidence$1: Ordering[T]`，这是上下文界定的特定编译方式，大家需要牢记这个编译规则；

- 上述函数`test1`中显式的添加了隐式参数`implicit ordering: Ordering[T]`，其原因是在编码阶段函数内部需要对`Ordering[T]`的实例对象进行调用，不得不添加该隐式参数（编译期动态插入的隐式参数在编码阶段引用不到），`test1`输入到`scala command`得到的结果为`test1: [T](first: T, second: T)(implicit evidence$1: Ordering[T], implicit ordering: Ordering[T])Unit`，在当前代码上下文中`main`函数调用`test1`的入参为`(Apple, Orange, FruitOrdering ,FruitOrdering) `，即单例对象`FruitOrdering`会同时出现在第三、四参数位上；

- 上述函数`test2`所处的上下文中并没有找到与`ClassTag`相对应的隐式值，这是`scala`编译器在编译时对`ClassTag`做特殊处理，在`scala command`下的编译的结果在预料之中，出现`ClassTag`的隐式参数，反编译字节码会发现`new Array[T](2)`被转换为`classTag.newArray(2)`，其内部通过调用`Java`中的`Array.newInstance(..)`动态创建数组对象；`main`函数中调用`test2`的代码的入参中隐式参数`ClassTag`被编译成`ClassTag..MODULE$.apply(Fruit.class)`，这是`scala`编译器的对`ClassTag`的特殊处理，大家明白是编译器行为即可。

### 类型约束

类型约束是一种比较严格的类型限定方式：

- `T =:= U` 表示T与U的类型相同
- `T <:< U` 表示T是U的子类型
- `T <%< U` 表示T可被隐式转换为U (`2.10 deprecated`,`2.11 removed`)

1. `T =:= U`是严格的类型约束，要求两个类型完全相等（包括类型参数），如：`List =:= List` is true，但是`List[Apple] =:= List[Orange]` is false；

2. `T <:< U`是严格的类型约束（与`<:`相比），要求前者`T`必须是后者`U`的子类或类型相同；

3. `T <%< U`在`scala 2.10`已经被标注deprecated，在`scala 2.11`被移除；同时视图界定`<%`从`scala 2.11`开始被标记为deprecated，在未来版本可能会被移除掉，在大家用到视图界定时候最好的选择是在需要隐式转换的地方进行显式的声明（可关注本文`视图界定`所讲述的内容）；

**注：** 类型约束顾名思义是为对类型进行约束的，仅此而已；

```scala

/**
  * restrict: T eq Orange
  */
def test0[T](i: T)(implicit ev: T =:= List[Fruit]): Unit = {
  i.foreach(println)
}

/**
  * same as [[test0()]]
  */
def test1[T](i: T)(implicit ev: =:=[T, Orange]): Unit = {
  println(i)
}

def test2[T](list: List[T])(implicit ev: T <:< Fruit): Unit = {
  val lis = new Fruit :: list
  lis.foreach(println)
}

```

**说明：**

- `=:=`, `<:<`实际上是两个在`Predef.scala`定义好的两个类（`sealed abstract class =:=[From, To] extends (From => To) with Serializable` 与 `sealed abstract class <:<[-From, +To] extends (From => To) with Serializable`），并在`scala`编译器的协助之下完成类型约束的行为，该行为发生于代码编译期间（感慨`scala`各种语法糖给编译器带来了大量的编译压力哈哈~~）；

- 上述函数`test0`, `test2`中隐式参数的类型是`scala`的中缀写法，原始写法如`test1`所示`=:=[A,B]`（带两个类型参数的类），当类有且仅有两个类型参数时候才能用中缀写法，更直观示例：`def foo(f: Function1[String, Int])`可以替换为中缀形式`def foo(f: String Function1 Int)`；

- `T <:< U`与`T <: U`的异同点说明，二者都表示`T`是`U`的子类，但`<:<`是更严格的类型约束， 要求在满足`T`是`U`子类的条件时不能对`T`做类型推导&隐式转换的操作；而`<:`则可以与类型推导&隐式转换配合使用；

  - **类型推导：**  

  下述代码在`main`函数中用`test3(1, List(new Apple))`调用`def test3[A, B <: A](a: A, b: B)`时编译器正常编译通过，调用`test3`时传入的第一个参数是`Int`，第二个参数是`List[Apple]`，显然不符合`B <: A`的约束，为了满足这个约束编译器在做类型推导时会继续向上寻找父类型来匹配是否满足，于是第一个参数被推导为`Any`类型，此时`List[Int]`符合`Any`的子类型，编译通过（用`Java Decompiler`反编译字节码发现`main`函数中调用`test3`的入参类型为`Int, List`，而这种入参类型不符合`test3`编译后的入参要求`B extends A`，即字节码合法性校验较编译器宽松许多，感觉需要读几本编译原理入门下，哈~）；

  ```scala

  def test3[A, B <: A](a: A, b: B): Unit = {
    println(a + ", " + b)
  }

  // 调用函数
  def main(args: Array[String]): Unit = {
    test3(1, List(new Apple))
  }

  ```  

  - **隐式转换：**

  下述代码中在调用`foo`时传入的`Apple`，`Orange`不能满足`Apple <: Orange`，但由于隐式函数`a2o`的存在，在编译`main`函数中的第一行 `foo(new Apple, new Orange)`时会引入隐式参数，将`Apple`转换为`Orange`，所以可成功编译并执行；而在编译第二行`bar(new Apple, new Orange)`会报错`error: Cannot prove that Apple <:< Orange`；

  ```scala

    implicit def a2o(a:Apple) = new Orange

    def foo[T, U<:T] (u:U, t:T) = print("OK")
    def bar[T, U](u:U, t:T)(implicit f: U <:< T) = println("OK")

    def main(args: Array[String]): Unit = {
      // 编译成功，引入隐式函数`a2o`
      foo(new Apple, new Orange)

      // 编译失败，`<:<`是严格类型限定
      // error: Cannot prove that Apple <:< Orange
      bar(new Apple, new Orange)
    }

  ```

### 协变（covariant）、 不变（invariance）、逆变（contravariance）

`scala`的`协变`、`逆变`、`不变`的特性相比较其他更为的难理解，计划单独新开一篇文详细对比讲解；感兴趣的小伙伴们可以移驾：

https://itinycheng.github.io/2019/01/30/scala-vs-java's-generics-2/

## 总结

`scala`类型参数体系是`scala`语言的重要特性，相比`Java`泛型，其更为复杂多变，且与编译器&隐式转换等互相掺杂配合，使得大家很难在短时间内掌握和灵活使用，建议大家多看`scala`源码，多练习，多尝试，多用`scala command`, `Java Decompiler`等工具测试&反编译二进制文件，以求更深刻的了解`scala`类型参数的语法特性&内在原理。

本文示例的代码存储在工程：https://github.com/itinycheng/jvm-lang-tutorial ，包`com.tiny.lang.java.generic` 和 `com.tiny.lang.scala.generic`中。

## 参考

- [https://stackoverflow.com/questions/3427345/what-do-and-mean-in-scala-2-8-and-where-are-they-documented](https://stackoverflow.com/questions/3427345/what-do-and-mean-in-scala-2-8-and-where-are-they-documented)
- [https://www.originate.com/thinking/stories/cheat-codes-for-contravariance-and-covariance/](https://www.originate.com/thinking/stories/cheat-codes-for-contravariance-and-covariance/)
- [https://www.clear.rice.edu/comp310/JavaResources/generics/co_contra_host_visitor.html](https://www.clear.rice.edu/comp310/JavaResources/generics/co_contra_host_visitor.html)
- [https://typelevel.org/blog/2016/02/04/variance-and-functors.html](https://typelevel.org/blog/2016/02/04/variance-and-functors.html)
- [https://docs.scala-lang.org/tour/variances.html](https://docs.scala-lang.org/tour/variances.html)
- [http://hongjiang.info/scala-type-contraints-and-specialized-methods/](http://hongjiang.info/scala-type-contraints-and-specialized-methods/)
- [https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html](https://docs.scala-lang.org/tutorials/FAQ/context-bounds.html)
- [https://www.zhihu.com/question/35339328](https://www.zhihu.com/question/35339328)
- [https://stackoverflow.com/questions/2723397/what-is-pecs-producer-extends-consumer-super](https://stackoverflow.com/questions/2723397/what-is-pecs-producer-extends-consumer-super)
