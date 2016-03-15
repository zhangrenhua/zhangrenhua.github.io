---
title: java设计模式-结构模式之代理模式
description: 代理（Proxy）模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。并且可以在不改变目标对象的情况下添加一些额外的功能。
tags: java设计模式
categories: java
date: 2015-11-07 17:11:12
---

## 背景

代理（Proxy）模式是对象的结构模式。代理模式给某一个对象提供一个代理对象，并由代理对象控制对原对象的引用。并且可以在不改变目标对象的情况下添加一些额外的功能。

## 模式中包含的角色及其职责
Subject：抽象主题角色，抽象主题类可以是抽象类，也可以是接口，是一个最普通的业务类型定义，无特殊要求。
RealSubject：具体主题角色，也叫被委托角色、被代理角色。是业务逻辑的具体执行者。
Proxy：代理主题角色，也叫委托类、代理类。它把所有抽象主题类定义的方法给具体主题角色实现，并且在具体主题角色处理完毕前后做预处理和善后工作。（最简单的比如打印日志）

### 代码示例
Subject：
```
// 抽象主题，定义主要功能=
publicinterface Subject {
   publicvoid operate();
}
```

RealSubject：
```
// 具体主题
publicclass RealSubject implements Subject{
 
   @Override
   publicvoid operate() {
        System.out.println("realsubject operatestarted......");
   }
}
```

Proxy：
```
// 代理类
publicclass Proxy implements Subject{
 
   private Subject subject;
 
   public Proxy(Subject subject) {
        this.subject = subject;
   }
 
   @Override
   publicvoid operate() {
        System.out.println("before operate......");
        subject.operate();
        System.out.println("after operate......");
   }
}
```

Client：
```
// 客户端
publicclass Client {
   /**
    * @param args
    */
   publicstaticvoid main(String[] args) {
        Subject subject = new RealSubject();
        Proxy proxy = new Proxy(subject);
        proxy.operate();
   }
}
```
运行结果：
eforeoperate......
realsubject operate started......
afteroperate......

## 应用场景
现实世界中，秘书就相当于一个代理，老板开会，那么通知员工开会时间、布置会场、会后整理会场等等开会相关工作就可以交给秘书做，老板就只需要开会就行了，不需要亲自做那些事。同理，在我们程序设计中也可使用代理模式来将由一系列无关逻辑组合在一起的代码进行解耦合，比如业务代码中的日志代码就可以在代理中进行。Spring的AOP就是典型的动态代理应用。

## 代理模式的应用形式
1、**远程代理(Remote Proxy)**
可以隐藏一个对象存在于不同地址空间的事实。也使得客户端可以访问在远程机器上的对象，远程机器可能具有更好的计算性能与处理速度，可以快速响应并处理客户端请求。
2、**虚拟代理(Virtual Proxy)**
允许内存开销较大的对象在需要的时候创建。只有我们真正需要这个对象的时候才创建。
3、**写入时复制代理(Copy-On-Write Proxy)**
用来控制对象的复制，方法是延迟对象的复制，直到客户真的需要为止。是虚拟代理的一个变体。
4、**保护代理(Protection (Access)Proxy)**
为不同的客户提供不同级别的目标对象访问权限
5、**缓存代理(Cache Proxy)**
为开销大的运算结果提供暂时存储，它允许多个客户共享结果，以减少计算或网络延迟。
6、**防火墙代理(Firewall Proxy)**
控制网络资源的访问，保护主题免于恶意客户的侵害。
7、**同步代理(SynchronizationProxy)**
在多线程的情况下为主题提供安全的访问。
8、**智能引用代理(Smart ReferenceProxy)**
当一个对象被引用时，提供一些额外的操作，比如将对此对象调用的次数记录下来等。
9、**复杂隐藏代理(Complexity HidingProxy)**
用来隐藏一个类的复杂集合的复杂度，并进行访问控制。有时候也称为外观代理(Façade Proxy)，这不难理解。复杂隐藏代理和外观模式是不一样的，因为代理控制访问，而外观模式是不一样的，因为代理控制访问，而外观模式只提供另一组接口。






