---
title: 在 Java 中替换大量 if-else 分支的方法
date: 2019-08-15 13:23:04
tags:
  - Java
  - 代码风格
---
## 问题描述
 在日常的开发中，有时会写出一些功能正常但不够简洁优雅的代码，例如，滥用 if-else。

### 案例一
在某个业务场景中，需要联合多个数据源查询数据，因为同时涉及到关系型和非关系型数据库，无法通过单一的 SQL 实现，只能分别去查。

于是，问题就出现了。由于是用前一个数据库查到的数据作为条件去另一个数据库查询，在执行之前就需要先判断一下是否为 null（为了使代码显得优雅，这里可以使用` Optional` 类代替 `xx == null`这样的语句。），结果就是出现了令人厌恶的多重嵌套分支：

```
if (!list.isEmpty()) {
    ...
    if (optional.isPresent()) {
        ...
        if (field != null) {
            ...
            if (...) {
                ...
            }
        }
    }
}
```

### 案例二
 有时也经常遇到这样的情况：要根据不同条件执行不同的操作，例如根据某变量的取值决定执行的代码：

```
if (a == xxx) {
    ...
}else if {
    ...
}else if {
    ...
}else if {
...
```

 当然，我们完全可以使用 switch case 结构来实现这样的功能，但是 switch 语句过长仍然会导致可读性和扩展性差，违背程序设计的开闭原则。甚至，在业务变更的时候，switch case 比 if else 更加难以进行修改。

## 优化思路
 很显然，这样的代码实在过于丑陋。

 正好，我目前在读阿里巴巴的孤尽大神所著的《码出高效：Java 开发手册》这本书，里面有一段就是描述了这种情况：

> 多重嵌套不能超过三层。多层嵌套在哪里都不受欢迎，是因为条件判断和分支逻辑数量呈指数关系。如果非得使用多层嵌套，请使用状态设计模式。对于超过3层的 if-else 的逻辑判断代码，可以使用卫语句、策略模式、状态模式等来实现

 既然大神给提供了思路，那就试着去做呗！根据我脑子里那点可怜的 Java 基础，艰难地归纳了一下这几种方案的具体实现。

## 方案实现
### 1. 卫语句
 所谓 **卫语句(guard clauses)** 就是把复杂的条件表达式拆分成多个条件表达式。例如在案例一中，很多的判断其实只是为了确定是否需要执行下一步，那么就可以在判断不满足条件后直接 return。修改后的结构如下：

```
if (list.isEmpty()) {
    return null;
}
...
if (!optional.isPresent()) {
    return null;
}
...
if (field == null) {
    return null;
}
...
if (...) {
    ...
}
```

### 2. 策略模式
**策略模式（Strategy Pattern）** 属于行为型的设计模式，其用意是针对一组算法，将每一个算法封装到具有共同接口的独立的类中，从而使得它们可以相互替换。策略模式使得算法可以在不影响到客户端的情况下发生变化。

 例如 switch 语句的每个不同 case 分支可以看作是执行不同策略，那么将这些策略抽取出来，封装到类中，然后定义一个公共的接口以供调用，就可以用一段调用代码替换掉整个分支结构。

 要使用策略模式，首先就是要定义调用的接口：

```
public interface Strategy {
    public void execute();
}
```

 然后，定义具体策略实现类：

```
public class StrategyA implements Strategy {

    @override
    public void execute() {
        ... //do something
    }
}

public class StrategyB implements Strategy {
    @override
    public void execute() {
        ... //do something
    }
}
public class StrategyC implements Strategy {
    @override
    public void execute() {
        ... //do something
    }
}
```


最后，定义 context对象以调用实际策略：

```
public class Context {

    Strategy strategy;

    public Context(Strategy strategy) {
        this.strategy = strategy;
    }
    
    public execute() {
        this.strategy.execute();
    }
}
```

 这里注意，需要传入对应的实例对象以调用它的方法，一般其它教程会直接创建或者使用简单工厂模式以获得对象，例如：

```
public class contextFactory {

    public getStrategy(String strategyStr) {
        switch (strategyStr) {
            case "A" : return new StrategyA();
            case "B" : return new StrategyB();
            case "C" : return new StrategyC();
            ...
        }
    }
}
```

 发现没有？这样又重新引入了 if 或者 switch，这就与我们的初衷相违背了。那么，该如何避免呢？于是我想到了反射（reflect）。使用反射，我们可以直接通过策略名称获得相应的对象，修改工厂类代码如下：

```
public class ContextFactory {

    public getStrategy(String strategyStr) {
        return (Strategy) Class.forName("Strategy" + strategyStr); //根据类名返回对应实例
    }
}
```

 在实际代码中使用：

```
String strategyStr;
ContextFactory contextFactory = new ContextFactory();
...
try {	//注意加 try catch 块
    Strategy strategy = contextFactory.getStrategy(strategyStr); //获取策略对象
    strategy.execute(); //执行具体操作
}catch (Exception e) {
    e.printStackTrace();
}
...
```

### 3. 状态模式
待补充