---
title: java设计模式-结构模式之缺省适配器模式
description: 缺省适配器模式为一个接口提供缺省实现，这样子类型可以从这个缺省实现进行扩展，而不必从原有接口进行扩展。作为适配器模式的一个特例，缺省适配器模式在Java语言中有着特殊的应用。
tags: "java设计模式"
#tags：[hexo, next]
categories: java
date: 2015-11-07 17:11:12
---

## 背景

缺省适配器模式为一个接口提供缺省实现，这样子类型可以从这个缺省实现进行扩展，而不必从原有接口进行扩展。作为适配器模式的一个特例，缺省适配器模式在Java语言中有着特殊的应用。
## 概述
默认适配模式的使用情况：当一个类B需要使用接口A中的部分方法时，可以定义一个抽象类C将接口A中的方法平庸实现，然后将类B从抽象类C继承，重写类B需要的方法。
### 举例说明
如下列代码所示，存在一个接口InterfaceA，类ClassB只需要实现接口中的方法methodA，类classC只需要实现接口中的方法methodC。
```
// 默认适配模式（接口）
public interface InterfaceA {
    //方法A
    public void methodA();
    //方法B
    public void methodB();
    //方法C
    public void methodC();
}

// 默认适配模式（抽象类）,默认实现
public abstract class AdapterC implements InterfaceA {
    //方法A空实现
    public void methodA() {}
    //方法B空实现
    public void methodB() {}
    //方法C空实现
    public void methodC() {}

}
```
上面AdapterC类默认实现InterfaceA中所有方法。
```
// 实现类B
public class ClassB extends AdapterC {
    //重写方法methodA
    public void methodA() {
       System.out.println("实现接口A中的方法methodA");
    }
}

// 实现类C
public class ClassC extends AdapterC {
    //重写方法methodC
    public void methodC() {
        System.out.println("实现接口A中的方法methodC");
    }  
}
```

上面给出的是目标角色的源代码，这个角色是以一个JAVA接口的形式实现的。可以看出，这个接口声明了两个方法：sampleOperation1()和sampleOperation2()。而源角色Adaptee是一个具体类，它有一个sampleOperation1()方法，但是没有sampleOperation2()方法。
适配器角色Adapter扩展了Adaptee,同时又实现了目标(Target)接口。由于Adaptee没有提供sampleOperation2()方法，而目标接口又要求这个方法，因此适配器角色Adapter实现了这个方法。
```
public class Adapter extends Adaptee implements Target {
    /**
     * 由于源类Adaptee没有方法sampleOperation2()
     * 因此适配器补充上这个方法
     */
    @Override
    public void sampleOperation2() {
        //写相关的代码
    }

}
```
## 总结
### 什么情况下使用本模式
在任何时候，如果不准备实现一个接口的所有方法时，就可以模仿上面的做法制造一个抽象类，给出所有方法的平庸的具体实现。这样，从这个抽象类再继承下去的子类就不必实现所有的方法了。这时候，新的类就是Adapter模式中的Adapter类，而给出平庸实现的抽象类就可以被当做Adaptee类。
适配器模式把一个类的接口变换成客户端所期待的另一种接口。适配器模式时原本无法在一起工作的两个类能够在一起工作。如前所述，适配器模式的”平庸化“形式可以时所考察的类不必实现不需要的那部分接口。
