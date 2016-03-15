---
title: java设计模式-创建模式之类创建
description: 还记得前几年买的一本很不错的书《JAVA与模式》，当时匆匆忙忙看了一遍。前几天刚学会在github上写blog，所以想将java设计模式的种种详细记载下来，供后人学习以及自己温故知新。下面我会对创建模式的每种模式的实现、优缺点、应用进行详解。
tags: "java设计模式"
#tags：[hexo, next]
categories: java
date: 2015-11-07 17:11:12
---

## 背景

还记得前几年买的一本很不错的书《JAVA与模式》，当时匆匆忙忙看了一遍。前几天刚学会在github上写blog，所以想将java设计模式的种种详细记载下来，供后人学习以及自己温故知新。下面我会对创建模式的每种模式的实现、优缺点、应用进行详解。
创建模式（Creational Pattern）是对类的实例化过程的抽象化。一些系统在创建对象时，需要动态的决定怎么创建对象，创建那些对象，以及如何组合和表示这些对象。创建模式描述了怎样构建和封装这些动态的决定。
创建模式分为类的创建模式和对象的创建模式两种。
1、类的创建模式
类的创建模式使用继承关系，把类的创建延迟到子类，从而封装了客户端将得到哪些具体类的信息，并且隐藏了这些类的实例是如何被创建和放在一起的。
2、对象的创建模式
而对象的创建模式则把对象的创建过程动态地委派给另一个对象，从而动态地决定客户端将得到哪些具体类的实例，以及这些类的实例是如何被创建和组合在一起的。

`类`的创建模式包括：简单工厂模式、工厂方法模式、抽象工厂模式
`对象`的创建模式包含：单例模式、多例模式、建造模式、原始模型模式等。
本文对`类`的创建模式进行详解。
## 工厂模式
工厂模式专门负责将大量有共同接口的类实例化。工厂模式可以动态决定将哪一个类实例化，不必事先知道每次要实例化哪一个类。工厂模式有一下几种形态：
1、简单工厂（Simple Factory）模式：又称静态工厂方法模式（Static Factory Method Pattern）。
2、工厂方法（Factory Method）模式：又称多态性工厂（Polymorphic Factory）模式或虚拟构造子模式
3、抽象工厂（Abstract Factory）模式：又称工具箱（Kit或Toolkit）模式。
### 简单工厂模式
简单工厂模式是类的创建模式，又叫做静态工厂方法（Static Factory Method）模式。简单工厂模式是由一个工厂对象决定创建出哪种产品类的实例。
简单工厂方法有如下三种实现：`普通工厂方法`、`多个工厂方法`、`静态工厂方法`
#### 普通工厂方法
普通工厂（Simple Factory）模式，就是建立一个工厂类，对实现了同一接口的一些类进行实例的创建。
举例如下：（发送邮件）
首先，创建统一接口：
```
public interface Sender {
    public void Send();
}
```
其次，创建2个实现类：
```
public class MailSender implements Sender {
    @Override
    public void Send() {
        System.out.println("this is mailsender!");
    }
}

public class SmsSender implements Sender {

    @Override
    public void Send() {
        System.out.println("this is sms sender!");
    }
}
```
最后，创建工厂类：
```
public class SendFactory {

    public Sender produce(String type) {
        if ("mail".equals(type)) {
            return new MailSender();
        } else if ("sms".equals(type)) {
            return new SmsSender();
        } else {
            System.out.println("找不到对应的类型！");
            return null;
        }
    }
}

// 测试如下
public class FactoryTest {

    public static void main(String[] args) {
        SendFactory factory = new SendFactory();
        Sender sender = factory.produce("sms");
        sender.Send();
    }
}
```
输出：this is sms sender!
#### 多个工厂方法
多个工厂方法，是对普通工厂方法模式的改进，在普通工厂方法模式中，如果传递的字符串出错，则不能正确创建对象，而多个工厂方法模式是提供多个工厂方法，分别创建对象。
```
// 工厂类
public class SendFactory {
    
    public Sender produceMail(){
        return new MailSender();
    }
    
    public Sender produceSms(){
        return new SmsSender();
    }
}

// 测试类如下
public class FactoryTest {

    public static void main(String[] args) {
        SendFactory factory = new SendFactory();
        Sender sender = factory.produceMail();
        sender.Send();
    }
}
```
输出：this is mailsender!
#### 静态工厂方法
静态工厂方法，将上面的多个工厂方法模式里的方法置为静态的，不需要创建实例，直接调用即可。
```
// 工厂类
public class SendFactory {
    
    public static Sender produceMail(){
        return new MailSender();
    }
    
    public static Sender produceSms(){
        return new SmsSender();
    }
}

// 测试类
public class FactoryTest {

    public static void main(String[] args) {    
        Sender sender = SendFactory.produceMail();
        sender.Send();
    }
}
```
输出：this is mailsender!
#### 优缺点
优点：在简单工厂模式中，一个工厂类处于对产品类实例化的中心位置上，它知道每一个产品，它决定哪一个产品类应当被实例化。这个模式的优点是客户端相对独立的于产品创建的过程，并且在系统引入新产品的时候无需修改客户端，也就是说，它在某种程度上支持“开-闭”原则。

缺点：这个模式的缺点是对“开-闭”原则的支持不够，因为如果有新的产品加入到系统中去，就需要修改工厂类，将必要的逻辑加入到工厂类中。
总结：`工厂模式`适合凡是出现了大量的对象需要创建，并且具有共同的接口时，可以通过工厂方法模式进行创建。在以上的三种模式中，第一种如果传入的字符串有误，不能正确创建对象，第三种相对于第二种，不需要实例化工厂类，所以，大多数情况下，我们会选用`第三种——静态工厂方法模式`。
### 工厂方法模式
工厂方法（Factory Method）模式是简单工厂模式的进一步抽象和推广。由于使用了多态性，保持了简单工厂模式的优点，而且克服了它的缺点。
工厂方法模式，创建一个工厂接口和创建多个工厂实现类，这样一旦需要增加新的功能，直接增加新的工厂类就可以了，不需要修改之前的代码。
#### 举例说明
统一接口：
```
public interface Sender {
    public void Send();
}
```
两个实现类：
```
public class MailSender implements Sender {
    @Override
    public void Send() {
        System.out.println("this is mailsender!");
    }
}

public class SmsSender implements Sender {

    @Override
    public void Send() {
        System.out.println("this is sms sender!");
    }
}
```
统一工厂接口类：
```
public interface Provider {
    public Sender produce();
}
```
两个工厂类：
```
public class SendMailFactory implements Provider {
    
    @Override
    public Sender produce(){
        return new MailSender();
    }
}

public class SendSmsFactory implements Provider{

    @Override
    public Sender produce() {
        return new SmsSender();
    }
}
```
测试类：
```
public class Test {

    public static void main(String[] args) {
        Provider provider = new SendMailFactory();
        Sender sender = provider.produce();
        sender.Send();
    }
}
```
总结：工厂方法（Factory Method）模式的好处就是，如果你现在想增加一个功能：发及时信息，则只需做一个实现类，实现Sender接口，同时做一个工厂类，实现Provider接口，就可以了，无需去改动现成的代码。这样做，拓展性较好！
### 抽象工厂模式
抽象工厂模式是所有形态的工厂模式中最为抽象和最具一般性的一重形态。抽象工厂模式是工厂方法模式的升级版本，他用来创建一组相关或者相互依赖的对象。他与工厂方法模式的区别就在于，工厂方法模式针对的是一个产品等级结构；而抽象工厂模式则是针对的多个产品等级结构。在编程中，通常一个产品结构，表现为一个接口或者抽象类，也就是说，工厂方法模式提供的所有产品都是衍生自同一个接口或抽象类，而抽象工厂模式所提供的产品则是衍生自不同的接口或抽象类。
#### 举例说明
```
// 产品
interface IProduct1 {
    public void show();
}
interface IProduct2 {
    public void show();
}

class Product1 implements IProduct1 {
    public void show() {
        System.out.println("这是1型产品");
    }
}
class Product2 implements IProduct2 {
    public void show() {
        System.out.println("这是2型产品");
    }
}
// 抽象工厂
interface IFactory {
    public IProduct1 createProduct1();
    public IProduct2 createProduct2();
}
class Factory implements IFactory{
    public IProduct1 createProduct1() {
        return new Product1();
    }
    public IProduct2 createProduct2() {
        return new Product2();
    }
}
// 客户端
public class Client {
    public static void main(String[] args){
        IFactory factory = new Factory();
        factory.createProduct1().show();
        factory.createProduct2().show();
    }
}
```
#### 优缺点
优点：抽象工厂模式除了具有工厂方法模式的优点外，最主要的优点就是可以在类的内部对产品族进行约束。所谓的产品族，一般或多或少的都存在一定的关联，抽象工厂模式就可以在类内部对产品族的关联关系进行定义和描述，而不必专门引入一个新的类来进行管理。

缺点：产品族的扩展将是一件十分费力的事情，假如产品族中需要增加一个新的产品，则几乎所有的工厂类都需要进行修改。所以使用抽象工厂模式时，对产品等级结构的划分是非常重要的。

总结：无论是简单工厂模式，工厂方法模式，还是抽象工厂模式，他们都属于工厂模式，在形式和特点上也是极为相似的，他们的最终目的都是为了解耦。在使用时，我们不必去在意这个模式到底工厂方法模式还是抽象工厂模式，因为他们之间的演变常常是令人琢磨不透的。经常你会发现，明明使用的工厂方法模式，当新需求来临，稍加修改，加入了一个新方法后，由于类中的产品构成了不同等级结构中的产品族，它就变成抽象工厂模式了；而对于抽象工厂模式，当减少一个方法使的提供的产品不再构成产品族之后，它就演变成了工厂方法模式。
所以，在使用工厂模式时，只需要关心降低耦合度的目的是否达到了。