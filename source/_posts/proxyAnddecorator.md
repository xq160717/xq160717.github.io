---
title: 代理模式&装饰器模式
date: 2019-02-25 11:35:10
categories: 设计模式
toc: true
---

### 代理模式

代理模式是一个相对比较简单的设计模式，直接来看代码

```java
public interface Withdraw {
    void withdraw();
}

public class UserWithdraw implements Withdraw {

    @Override
    public void withdraw() {

    }
}

public class UserWithdrawProxy implements Withdraw {

    private UserWithdraw userWithdraw;

    UserWithdrawProxy() {
        this.userWithdraw = new UserWithdarw;
    }

    @Override
    public void withdraw() {
        System.out.println("被代理之前可以做一些其他操作");
        userWithdraw.withdraw();
    }
}

// 调用方
public class App {
    public static void main(String[] args) {
        Withdraw withdraw = new UserWithdrawProxy();
        withdraw.withdraw();
    }
}
```

代码很简单，两个类实现同一个接口，通过使用组合的形式持有对象引用，在代理类中去调用实际的实现，在真正进行操作之前可以做一些其他的事情，以下是代理模式的类图

<img src="proxy.png">

### 装饰器模式

装饰器模式，动态的给一个对象添加一些额外的职责，以对客户端透明的方式拓展对象的功能，是继承关系的一种替代方案。直接来看代码

```java
public interface ComponentInterface {

    void sayHello();

}

public class ComponentInterfaceImpl implements ComponentInterface {

    @Override
    public void sayHello() {
        System.out.println("默认实现");
    }
}

/**
 * 装饰器模板，为了提高代码的复用，简化具体装饰类的实现成本，当然不用的话可以删掉，直接实现接口
 */
public abstract class AbstractDecorator implements ComponentInterface {

    public abstract void sayHello();

}

public class ContreteDecoratorA extends AbstractDecorator {

    private ComponentInterface componentInterface;

    ContreteDecoratorA(ComponentInterface componentInterface) {
        this.componentInterface = componentInterface;
    }

    @Override
    public void sayHello() {
        System.out.println("默认实现前置增强");
        componentInterface.sayHello();
        System.out.println("默认实现后置增强");
    }
}

public class App {

    public static void main(String[] args) {
        ComponentInterface componentInterface = new ContreteDecoratorA(new ComponentInterfaceImpl());
        componentInterface.sayHello();
    }

}
```

AbstractDecorator为一个装饰器模板，目的是为了提高代码复用，简化具体装饰器子类的实现成本，不需要的话也是可以省略的，以下是装饰器模式的类图

<img src="decorate.png">

### 区别

从类图上看，代理模式和装饰器模式看起来非常相像。对装饰器模式来说，装饰者和被装饰者都实现同一个接口。对于代理模式来说，代理类和被被代理类也都实现同一个接口。此外，不管我们使用其中的哪一个模式，都很容地在真实对象的方法前面或者后面加上自定义的方法。

然而从实际上来说，装饰器模式和代理模式之间还是有很多差别的。装饰器模式关注于在一个对象上动态的添加方法，而代理模式关注于控制对对象的访问。换句话说就是，用代理模式，代理类可以对它的客户端隐藏一个对象的具体信息。因此，当使用代理模式的时候，我们常常在一个代理类中创建一个对象的实例。并且当我们使用装饰器模式的时候，我们通常的做法是将原始的对象作为一个参数传给装饰者的构造器。

我们可以从另一个角度来总结这些差别：使用代理模式，代理和真实对象之间的关系通常是在编译时就已经确定了，而装饰器模式能够在运行时被构造所需要的对象。

