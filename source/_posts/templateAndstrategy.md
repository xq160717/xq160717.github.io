---
title: 模板方法模式&策略模式
date: 2019-02-14 16:33:40
categories: 设计模式
toc: true
---

### 模板方法模式

模板方法模式本质就是类的继承，一个操作中的算法框架在父类中定义好，给出一个顶级的逻辑骨架，而将一些步骤的实现延迟到子类中。因此一个标准的模板方法一般是一个抽象类加上具体的实现子类，抽象类负责整个流程的定义，子类负责某一个具体流程的实现策略。下面是一个非常简单的模板方法的类图

<img src="template.png">

父类中定义了通知内容生成的抽象方法以及发送的抽象方法，具体的实现由子类EmailNotify以及SmsNotify去实现。

#### AQS中的模板方法模式

模板方法模式在框架中的应用非常广泛，这里我们可以看下在模板方法在AQS中的应用

<img src="aqs.png">

其中tryAcquire以及tryRelease等方法均需子类去实现，实现自定义同步组件时，将会调用同步器提供的模板方法，我们来看一下在ReentrantLock中的具体实现，下面是公平锁的实现(非公平锁同理)。

<img src="fairsync.png">

在调用lock方法的时候，会去调用acquire方法，acquire在父类中会继续调用子类的具体实现，也就是tryAcquire方法。

总结来说，模板方法类作为父类，其要做的事情就是针对需求进行规划，自己实现一部分，然后把需求拆分成更加细小的任务延迟到子类中去实现，这就是模板的责任以及目的。

### 策略模式

策略模式是一种比较简单的设计模式，但是在业务开发中会比较常见，当你的业务面对不同的场景需要执行不同的策略时，那么使用策略模式可以更好的写出低耦合的代码。策略模式的本质是将每个算法封装到具有公共接口的一系列策略类中，从而使它们可以互相替换，并且可以在不影响到调用方的情况下发现变化，策略模式仅仅封装算法，但策略模式并不决定在何时何地使用何种算法，算法的选择由调用方决定。

#### 类图

<img src="strategy.jpg">

- **Strategy**: 策略接口或者策略抽象类
- **ConcreateStrategyA**等：实现策略接口的具体策略类
- **Context**：上下文类，持有具体策略类的实例，并负责调用相关的算法

#### 实现

**Strategy接口**

策略接口，定义策略的执行

```java
public interface DragonSlayingStrategy {

    void execute();

}
```

**具体策略实现类**

```java
public class MeleeStrategy implements DragonSlayingStrategy {

    @Override
    public void execute() {
        System.out.println("近战攻击......");
    }
}

public class ProjectileStrategy implements DragonSlayingStrategy {

    @Override
    public void execute() {
        System.out.println("远程攻击......");
    }
}
```

**Context类**

Context类，持有具体策略的实例，负责调用具体的算法

```java
public class DragonSlayer {

    private DragonSlayingStrategy strategy;

    public DragonSlayer(DragonSlayingStrategy strategy) {
        this.strategy = strategy;
    }

    public void changeStrategy(DragonSlayingStrategy strategy) {
        this.strategy = strategy;
    }

    public void goToBattle() {
        strategy.execute();
    }
}
```

**测试结果**

```java
public class App {

    public static void main(String[] args) {

        DragonSlayer dragonSlayer = new DragonSlayer(new MeleeStrategy());
        dragonSlayer.goToBattle();

        dragonSlayer.changeStrategy(new ProjectileStrategy());
        dragonSlayer.goToBattle();

        dragonSlayer.changeStrategy(new SpellStrategy());
        dragonSlayer.goToBattle();
    }

}
```

由上述例子我们不难发现，客户端调用的时候可以根据自己的需要去调用不同的策略而不用关心策略的具体实现，而且策略类都实现的是同一个接口可以非常自由的进行切换，增加一个新的策略只需要添加一个具体的策略类即可，基本不需要改变原有的代码，符合开闭原则。

当我们的系统里有很多类，它们之间的区别仅在于它们的行为的时候，那么使用策略模式可以动态地让一个对象在许多行为中选择一个行为。

### 策略模式和模板方法的区别

从上面的分析我们不难发现，在业务场景不复杂的情况下，好像使用模板或者策略模式都可以解决问题。但是当业务复杂的时候，它们之间的区别就展开了。如果我们封装的算法是"一整个算法"，那么使用策略模式比较好。如果我们封装的变化只是算法中的一部分，而且我们不希望客户端直接使用这些方法，那么应该使用模板方法模式。





