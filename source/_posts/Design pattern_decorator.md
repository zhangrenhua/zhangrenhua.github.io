---
title: java设计模式-结构模式之修饰模式
description: 装饰（Decorator）模式又名包装模式。装饰模式以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案。</br>装饰模式使用原来被装饰的类的一个子类的实例，把客户端的调用委派到被装饰类。装饰模式的关键在于这种扩展是完全透明的。
tags: java设计模式
categories: java
date: 2015-11-07 17:11:12
---

##  背景

装饰（Decorator）模式又名包装模式。装饰模式以对客户端透明的方式扩展对象的功能，是继承关系的一个替代方案。
装饰模式使用原来被装饰的类的一个子类的实例，把客户端的调用委派到被装饰类。装饰模式的关键在于这种扩展是完全透明的。

## 简介
Decorator装饰器，顾名思义，就是动态地给一个对象添加一些额外的职责，就好比为房子进行装修一样。因此，装饰器模式具有如下的特征：
1、它必须具有一个装饰的对象；
2、它必须拥有与被装饰对象相同的接口；
3、它可以给被装饰对象添加额外的功能。
装饰器通过包装一个装饰对象来扩展其功能，而又不改变其接口，这实际上是基于对象的适配器模式的一种变种。它与对象的适配器模式的异同点如下。
相同点：都拥有一个目标对象。
不同点：适配器模式需要实现另外一个接口，而装饰器模式必须实现该对象的接口。
### 示例详解
定义接口函数operation()：
```
public interface Sourcable {
    public void operation();

}
```
下面Source是Sourcable的一个实现，其函数operation()负责往控制台输出一个字符串：
```
public class Source implements Sourcable {

    public void operation() {
        System.out.println("原始类的方法");
    }

}

```
装饰器类Decorator1采用了典型的对象适配器模式，它首先拥有一个Sourcable对象 source ，该对象通过构造函 数进行初始化。然后它实现了 Sourcable接口，以期保持与 source 同样的接口，并在重写的 operation() 函数中调用  source 的 operation() 函数，在调用前后可以实现自己的输出，这就是装饰器所扩展的功能：
```
public class Decorator1 implements Sourcable {

    private Sourcable sourcable;
    public Decorator1(Sourcable sourcable){
        super();
        this.sourcable=sourcable;
    }
    
    public void operation() {
        System.out.println("装饰器前");
        sourcable.operation();
        System.out.println("装饰器后");

    }

}

```
装饰器类Decorator2是另一个装饰器，不同的是它装饰的内容不一样，即输出了不同的字符串:
```
public class Decorator2 implements Sourcable {

    private Sourcable sourcable;
    public Decorator2(Sourcable sourcable){
        super();
        this.sourcable=sourcable;
    }
    public void operation() {
        System.out.println("第二个装饰器前");
        sourcable.operation();
        System.out.println("第二个装饰器后");

    }

}
```
这时，我们就可以像使用对象的适配器模式一样来使用这些装饰器，使用不同的装饰器就可以达到不同的装饰效果:
```
public class DecoratorTest {

    /**
     * @param args
     */
    public static void main(String[] args) {
        Sourcable source = new Source();

        // 装饰类对象 
        Sourcable obj = new Decorator1(new Decorator2(source));
        obj.operation();
    }

}

```
## 总结
### 装饰模式的优点
装饰模式与继承关系的目的都是要扩展对象的功能，但是装饰模式可以提供比继承更多的灵活性。装饰模式允许系统动态决定“贴上”一个需要的“装饰”，或者除掉一个不需要的“装饰”。继承关系则不同，继承关系是静态的，它在系统运行前就决定了。

通过使用不同的具体装饰类以及这些装饰类的排列组合，设计师可以创造出很多不同行为的组合。
### 装饰模式的缺点
由于使用装饰模式，可以比使用继承关系需要较少数目的类。使用较少的类，当然使设计比较易于进行。但是，在另一方面，使用装饰模式会产生比使用继承关系更多的对象。更多的对象会使得查错变得困难，特别是这些对象看上去都很相像。
### 设计模式在JAVA I/O库中的应用
装饰模式在Java语言中的最著名的应用莫过于Java I/O标准库的设计了。
由于Java I/O库需要很多性能的各种组合，如果这些性能都是用继承的方法实现的，那么每一种组合都需要一个类，这样就会造成大量性能重复的类出现。而如果采用装饰模式，那么类的数目就会大大减少，性能的重复也可以减至最少。因此装饰模式是Java I/O库的基本模式。