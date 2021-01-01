---
title: 一次Java函数式编程实践
tags:
  - Java
  - 函数式编程
  - 代码风格
originContent: ''
categories: []
toc: true
date: 2020-10-14 10:56:32
---

# 一次Java函数式编程实践

## 1. 问题背景

在日常工作中，常会遇到一种场景：在一个方法中，我们需要根据不同的传入参数选择调用不同的方法。

最为简单粗暴的解决方式就是使用 if-else 或者switch 语句去分情况选择

```java
public void method(String type){
    if (type.equals("xxx")) {
        methodA();
    }else if (type.equals("yyy")) {
        methodB();
    }else if (...) {
        ...
    }
}
```

但是我个人是比较不喜欢这种写法的，可扩展性太差，不符合开闭原则。想要优化代码，我首先想到的是使用某种设计模式，例如策略模式。

可以定义一个统一的策略接口，编写不同实现类，然后定义 context 对象用于切换上下文。业务代码中只需要将参数传递给 context 对象去选择对应实现类就行。具体可以参考我另一篇博文：[在 Java 中替换大量 if-else 分支的方法](https://fly-bear.github.io/2019/08/15/replace-if/)

不过设计模式虽好，但在开发过程中是比较麻烦的，而且代码量增加了许多，显得有些笨重。很多时候只是一个小功能，不想引入设计模式，有没有更优雅的解决方式呢？

这时候我想到了时下流行的函数式编程。

## 2. 什么是函数式编程

### 简单描述

关于函数式编程的定义和本质，这个问题比较复杂，在这我以自己的理解做一些粗浅的描述。

所谓**函数式编程**（functional programming）指的是一种**编程范式**（programming paradigm），是编程的方法论，例如面向对象编程就是**命令式编程**（Imperative programming）的一种。

命令式编程是面向**计算机硬件**的抽象，有**变量**（对应着存储单元），**赋值语句**（获取，存储指令），**表达式**（内存引用和算术运算）和**控制语句**（跳转指令）。

函数式编程中的**函数**这个术语不是指计算机中的函数（Subroutine），而是指数学中的函数，即自变量的映射是面向数学的抽象，将计算描述为一种**表达式求值**。

不同于命令式编程的将计算过程一步步描述给计算机这种思想，函数式编程倾向于将程序写成各种表达式的集合。

举一个数学上的例子：

$h(g(f(x))) = (h*(g*f))(x) = ((h*g)*f)(x)$

命令式编程就类似于最左边的表达式，一个步骤返回数据后再给另一个步骤执行，而函数式编程就像右边两个表达式，程序表现为函数的交织，最后灌入数据得到结果即可。从这里就能体现出函数式编程的第一个特点：函数是“第一等公民”，和其它类型数据一样，允许作为参数和结果。

### 五个特点：

1. 函数是"第一等公民"

2. 只用"表达式"，不用"语句"。即每一步都是单纯的运算，而且都有返回值。

3. 没有"副作用"。意味着函数要保持独立，所有功能就是返回一个新的值，没有其他行为，尤其是不得修改外部变量的值。

4. 不修改状态。意味着状态不能保存在变量中，变量的值是不允许修改的。函数式编程使用参数保存状态，最好的例子就是递归。

5. 引用透明。指的是函数的运行不依赖于外部变量或"状态"，只依赖于输入的参数，任何时候只要参数相同，引用函数所得到的返回值总是相同的。

## 3. 代码优化

### Java 8 函数式接口

虽然上面说了这么多，但实际上我并不是想把代码完全改成函数式风格，只想利用它的一个特点——可以将函数作为参数。使用策略模式可以根据参数切换使用的函数，那如果能直接将使用的函数传递进去岂不是更简洁？

其实 Java 早就支持了函数式的编程，最典型的就是 1.8 的 lambda 表达式。除此之外日常编程中接触最多的还有一个，就是 `Stream` 类。`Stream` 中有许多方法可以直接以函数名作为参数，例如 `map()`方法:

```java
public final <R> Stream<R> map(Function<? super P_OUT, ? extends R> mapper) {
    Objects.requireNonNull(mapper);
    return new StatelessOp<P_OUT, R>(this, StreamShape.REFERENCE, StreamOpFlag.NOT_SORTED | StreamOpFlag.NOT_DISTINCT) {
        @Override
        Sink<P_OUT> opWrapSink(int flags, Sink<R> sink) {
            return new Sink.ChainedReference<P_OUT, R>(sink) {
                @Override
                public void accept(P_OUT u) {
                    downstream.accept(mapper.apply(u));
                }
            };
        }
    };
}
```

`Function`接口在 java.util 中定义，它指定了一个输入类型和一个输入类型（可以使用泛型），简单的方法传递可以直接将参数定义为 Function 类型：

```java
/**
* Represents a function that accepts one argument and produces a result.
*
* <p>This is a <a href="package-summary.html">functional interface</a>
* whose functional method is {@link #apply(Object)}.
*
* @param <T> the type of the input to the function
* @param <R> the type of the result of the function
*
* @since 1.8
*/

@FunctionalInterface
public interface Function<T, R> {

    R apply(T t);

}
```

但如果我们的方法有多个传入参数，那么 `Function`就不是很适用了。当然也有可以接受两个入参的 `BiFunction`，不过也并不太泛用。

这时候就需要我们自己实现一个函数式接口。

函数式接口(Functional Interface)是 Java8 引入的新特性，它是有且仅有一个抽象方法，但是可以有多个非抽象方法的接口，它可以被隐式转换为 lambda 表达式。函数式接口可以用`@FunctionalInterface`注解标记，在编译时会进行格式检查。

### 定义接口

假设我有一系列方法：

```java
public class MethodClass {

    String methodA(Long l, Integer i, Double b) {
        ...
    }

    String methodB(Long l, Integer i, Double b) {
        ...
    }

    ...
}
```

我需要在另一个方法中按传入条件调用它们中的一个，那么我可以先定义一个函数式接口：

```java
@FunctionalInterface

public interface MyFunction {

    String apply(Long l, Integer i, Double d);

}
```

将接口作为参数类型传递(定义为 static 是为了方便 main 函数调用)，在方法体中调用接口的 apply 方法：

```java
public static void method(MyFunction function, Long l, Integer i, Double d) {

    function.apply(l, i, d);

}
```

于是调用此方法时便可以用 lambda 表达式实现接口后传入：

```java
public static void main(String[] args) {
    MethodClass methodClass = new MethodClass();
    Long a = 1L;
    Integer b = 1;
    Double c = 0D;
    method((l, i, d) -> methodClass.methodA(l, i, d), a, b, c);
}
```

进一步简化可以写成：

```java
public static void main(String[] args) {
    MethodClass methodClass = new MethodClass();
    Long a = 1L;
    Integer b = 1;
    Double c = 0D;
    method(methodClass::methodA, a, b, c);
}
```
