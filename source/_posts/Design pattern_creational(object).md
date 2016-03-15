---
title: java设计模式-创建模式之对象创建
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
本文对`对象`的创建模式进行详解。
## 单例模式
作为对象的创建模式，单例模式确保某一个类只有一个实例，而且自行实例化并向整个系统提供这个实例。这个类成为单例类。
单例（Singleton）模式的三大要点：
一、某个类只能有一个实例；
二、它必须自行创建这个实例；
三、它必须自行向整个系统提供这个实例。
单例模式又有三种实现：懒汉式、饿汉式、登记式。以下对这三种实现进行详解.
### 懒汉式
示例:
```
//懒汉式单例类.在第一次调用的时候实例化自己 
public class Singleton {
    private Singleton() {}
    private static Singleton single=null;
    //静态工厂方法 
    public static Singleton getInstance() {
         if (single == null) {  
             single = new Singleton();
         }  
        return single;
    }
}
```
Singleton通过将构造方法限定为private避免了类在外部被实例化，在同一个虚拟机范围内，Singleton的唯一实例只能通过getInstance()方法访问。
（事实上，通过Java反射机制是能够实例化构造方法为private的类的，那基本上会使所有的Java单例实现失效。此问题在此处不做讨论，姑且掩耳盗铃地认为反射机制不存在。）
但是以上懒汉式单例的实现没有考虑线程安全问题，它是线程不安全的，并发环境下很可能出现多个Singleton实例，要实现线程安全，有以下三种方式，都是对getInstance这个方法改造，保证了懒汉式单例的线程安全。
1、在getInstance方法上加同步
```
public static synchronized Singleton getInstance() {
         if (single == null) {  
             single = new Singleton();
         }  
        return single;
}
```
2、双重检查锁定
```
public static Singleton getInstance() {
        if (singleton == null) {  
            synchronized (Singleton.class) {  
               if (singleton == null) {  
                  singleton = new Singleton(); 
               }  
            }  
        }  
        return singleton; 
    }
```
3、静态内部类
```
public class Singleton {  
    private static class LazyHolder {  
       private static final Singleton INSTANCE = new Singleton();  
    }  
    private Singleton (){}  
    public static final Singleton getInstance() {  
       return LazyHolder.INSTANCE;  
    }  
}  
```
第三种比前面二种都好一些，既实现了线程安全，又避免了同步带来的性能影响。
### 饿汉式
示例：
```
//饿汉式单例类.在类初始化时，已经自行实例化 
public class Singleton1 {
    private Singleton1() {}
    private static final Singleton1 single = new Singleton1();
    //静态工厂方法 
    public static Singleton1 getInstance() {
        return single;
    }
}
```
饿汉式在类创建的同时就已经创建好一个静态的对象供系统使用，以后不再改变，所以天生是线程安全的。
### 登记式单例
示例：
```
//类似Spring里面的方法，将类名注册，下次从里面直接获取。
public class Singleton3 {
    private static Map<String,Singleton3> map = new HashMap<String,Singleton3>();
    static{
        Singleton3 single = new Singleton3();
        map.put(single.getClass().getName(), single);
    }
    //保护的默认构造子
    protected Singleton3(){}
    //静态工厂方法,返还此类惟一的实例
    public static Singleton3 getInstance(String name) {
        if(name == null) {
            name = Singleton3.class.getName();
            System.out.println("name == null"+"--->name="+name);
        }
        if(map.get(name) == null) {
            try {
                map.put(name, (Singleton3) Class.forName(name).newInstance());
            } catch (InstantiationException e) {
                e.printStackTrace();
            } catch (IllegalAccessException e) {
                e.printStackTrace();
            } catch (ClassNotFoundException e) {
                e.printStackTrace();
            }
        }
        return map.get(name);
    }
    //一个示意性的商业方法
    public String about() {    
        return "Hello, I am RegSingleton.";    
    }    
    public static void main(String[] args) {
        Singleton3 single3 = Singleton3.getInstance(null);
        System.out.println(single3.about());
    }
}
```
`登记式`单例模式实际上维护了一组单例类的实例，将这些实例存放在一个Map（登记薄）中，对于已经登记过的实例，则从Map直接返回，对于没有登记的，则先登记，然后返回。 
另外其实`登记式`单例模式内部实现还是用的饿汉式单例，因为其中的static方法块，它的单例在类被装载的时候就被实例化了。
### 饿汉式和懒汉式区别
饿汉式：
1、饿汉就是类一旦加载，就把单例初始化完成，保证getInstance的时候，单例时已经存在的了。
2、饿汉式在类创建的同时就实例化一个静态对象出来，不管之后会不会使用这个单例，都会占据一定的内存，但是相应的，在第一次调用时速度也会更快，因为其资源已经初始化完成，
3、饿汉式天生就是线程安全的，可以直接用于多线程而不会出现问题，
懒汉式：
1、懒汉比较懒，只有当调用getInstance的时候，才回去初始化这个单例。
2、懒汉式会延迟加载，在第一次使用该单例的时候才会实例化对象出来，第一次调用时要做初始化，如果要做的工作比较多，性能上会有些延迟，之后就和饿汉式一样了。
3、懒汉式本身是非线程安全的，为了实现线程安全有几种写法，分别是上面的1、2、3，这三种实现在资源加载和性能方面有些区别。
### 结论
由结果可以得知单例模式为一个面向对象的应用程序提供了对象惟一的访问点，不管它实现何种功能，整个应用程序都会同享一个实例对象。
主要理解：对于单例模式的几种实现方式，饿汉式和懒汉式的区别，线程安全，资源加载的时机，还有懒汉式为了实现线程安全的3种方式的细微差别。
## 多例模式
所谓多例模式（Multiton Pattern），实际上就是单例模式的自然推广。作为对象的创建模式，多例模式或多例类有以下特点：
1、多例类可有多个实例
2、多例类必须自己创建、管理自己的实例，并向外界提供自己的实例。
### 举例说明
比如每一麻将局都有两个骰子，因此骰子就应当是双态类，下面就以这个系统为例，说明多例模式的结构。
```
import java.util.Date;
import java.util.Random;

public class Die {
    private static Die die1 = new Die();
    private static Die die2 = new Die();

    /**
     * 私有的构造函数保证外界无法直接将此类实例化
     *
     */
    private Die() {
    }

    /**
     * 工厂方法
     */
    public static Die getInstance(int whichOne) {
        if (whichOne == 1) {
            return die1;
        } else {
            return die2;
        }
    }

    /**
     * 投骰子，返回一个在1－6之间的随机书
     */
    public synchronized int dice() {
        Date d = new Date();
        Random r = new Random(d.getTime());
        int value = r.nextInt();// 获取随机数
        value = Math.abs(value);// 获取随机数的绝对值
        value = value % 6;// 对6取模
        value += 1;// 由于 value的范围是0－5，所以value+1成为1-6
        return value;
    }

    /**
     * 测试代码
     */
    public static void main(String[] args) {

        Die die1, die2;
        die1 = Die.getInstance(1);
        die2 = Die.getInstance(2);

        System.out.println(die1.dice());
        System.out.println(die2.dice());
    }
}
```
### 总结
多例模式有2中实现：一有上限多例模式，二无上限多例模式。

有上限的多例类：由于有上限的多例类对实例的数目有上限，因此有上限的多例类在这个上限等于1时，多例类就回到了单例类，因此多例类是单例类的推广，而单例类是多例类的特殊情况。
一个有上限的多例类可以使用静态变量存储所有的实例，特别时在实例数目不多的时候，可以使用一个个的静态变量存储一个个的实例。在数目较多的时候，就需要使用静态聚集存储这些实例。

无上限多例模式：无上限多例模式实例数目没有上限的多例模式叫无上限多例模式。
由于没有上限的多例类对实例的数目是没有限制的，因此，虽然这种多例模式是单例模式的推广，但是这种多例类并不一定能够回到单例类。
由于事先不知道要创建多少个实例，因此，必然使用聚集管理所有的实例。

案例：国际化解决方案
根据一个对象创建出多种语言的实例。
## 建造（Builder）模式
建造模式是对象的创建模式。建造模式可以将一个产品的内部表象与产品的成分过程分割开来，从而可以是一个建造过程生成具有不同的内部表象的产品对象。
与抽象工厂的区别：在建造者模式里，有个指导者，由指导者来管理建造者，用户是与指导者联系的，指导者联系建造者最后得到产品。即建造模式可以强制实行一种分步骤进行的建造过程。
### 举例说明
```
// 产品建造接口
public interface Builder {
    void buildPartA();

    void buildPartB();

    void buildPartC();

    Product getResult();
}
public interface Part {}
public interface Product {}

// 具体建造工具
public class ConcreteBuilder implements Builder {

    private Part partA, partB, partC;

    @Override
    public void buildPartA() {
        System.out.println("A");
    }

    @Override
    public void buildPartB() {
        System.out.println("B");
    }

    @Override
    public void buildPartC() {
        System.out.println("C");

    }

    @Override
    public Product getResult() {
        // 返回产品结果
        return null;
    }
}
// 建造者
public class Director {

    private Builder builder;

    public Director(Builder builder) {
        this.builder = builder;
    }

    public void construct() {
        builder.buildPartA();
        builder.buildPartB();
        builder.buildPartC();
    }
}

// 建造者
public class Director {

    private Builder builder;

    public Director(Builder builder) {
        this.builder = builder;
    }

    public void construct() {
        builder.buildPartA();
        builder.buildPartB();
        builder.buildPartC();
    }

    //测试代码
    public static void main(String[] args) {
        ConcreteBuilder builder = new ConcreteBuilder();
        Director director = new Director(builder);
        director.construct();
        Product product = builder.getResult();
    }
}
```
### 总结
使用建造者模式的好处
1、使用建造者模式可以使客户端不必知道产品内部组成的细节。
2、具体的建造者类之间是相互独立的，对系统的扩展非常有利。
3、由于具体的建造者是独立的，因此可以对建造过程逐步细化，而不对其他的模块产生任何影响。

使用建造者模式的场合:
1、创建一些复杂的对象时，这些对象的内部组成构件间的建造顺序是稳定的，但是对象的内部组成构件面临着复杂的变化。
2、要创建的复杂对象的算法，独立于该对象的组成部分，也独立于组成部分的装配方法时。
## 原始（Prototype）模式
原始模型模式属于对象的创建模式。通过给出一个原型对象来指明所要创建的对象的类型，然后用复制这个原型对象的方法创建出更多同类型的对象。这就是原始模型模式的用意。
原始模型模式分为简单形式和登记形式，下面对这两种实现详解。
### 原始模型模式-简单形式
如果需要创建的原型对象数目较少而且比较固定的话，可以采取第一种形式，也即简单形式的原始模型模式。在这种情况下，原型对象的引用可以由客户端自己保存。
示例：
```
// 原型定义
interface Prototype extends Cloneable {

    public Prototype clone();

}

// 原型实现
class ConcretePrototype implements Prototype {

    public Prototype clone() {
        try {
            return (Prototype) super.clone();
        } catch (CloneNotSupportedException e) {
            return null;
        }
    }

}

// 测试代码
public class Client {
    private Prototype prototype;

    public void operation(Prototype example) {
        this.prototype = example.clone();
        System.out.println(prototype);
    }

    public static void main(String[] args) {
        new Client().operation(new ConcretePrototype());
    }
}
```
### 原始模型模式-登记形式
如果要创建的原型对象数目不固定的话，可以采用登记形式，也即登记形式的原始模型模式。在这种情况下，客户端并不保存对原型对象的引用，这个任务交给管理员对象。在复制一个原型对象之前，客户端可以查看管理员对象是否已存在一个满足要求的原型对象。如果有，可以直接从管理员类取得对这个对象的引用；如果没有，客户端就要自己复制此原型对象。
示例：
```
// 原型定义
interface Prototype extends Cloneable {

    public Object clone();

}

// 原型实现
class ConcretePrototype implements Prototype {

    public synchronized Object clone() {
        Prototype temp = null;
        try {
            temp = (Prototype) super.clone();
        } catch (CloneNotSupportedException e) {
            System.out.println("Clone failed.");
        }
        return temp;
    }

}

// 原型实例管理器
class PrototypeManager {
    private Vector<Prototype> objects = new Vector<Prototype>();

    // 添加新对象
    public void add(Prototype prototype) {
        objects.add(prototype);
    }

    // 获取对象
    public Prototype get(int i) {
        return (Prototype) objects.get(i);
    }

    public int getSize() {
        return objects.size();
    }
}

// 测试代码
public class Client {
    private PrototypeManager mgr;
    private Prototype prototype;

    public void registrPrototype() {
        prototype = new ConcretePrototype();
        Prototype copyPrototype = (Prototype) prototype.clone();
        mgr.add(copyPrototype);
    }

    public static void main(String[] args) {
        new Client().registrPrototype();
    }

}
```
### clone
由于原始模型模式中底层使用`clone`，所以在此详细解释下。
java中复制或克隆有两种方式。这两种方式分别叫做浅复制（浅克隆）和深复制（深克隆）。<font color=#FF4500>原型模式模式中可以随意使用下面2种克隆方式。</font>
#### 浅复制（浅克隆）
被复制对象的所有变量都含有与原来的对象相同的值，而且所有的对其他对象的引用都仍然指向原来的对象。换言之，浅复制仅仅复制所考虑的对象，而不复制它所引用的对象。
示例：
```
class Wife implements Serializable {
    String name = "范冰冰";
    String birthday = "20151115";

    @Override
    public String toString() {
        return "Wife [name=" + name + ", birthday=" + birthday + "]";
    }
}

class Husband implements Cloneable, Serializable {
    Wife wife = new Wife();
    String birthday = "20151115";

    // 浅克隆一个对象
    public Object clone() {
        Husband husband = null;
        try {
            husband = (Husband) super.clone();
        } catch (CloneNotSupportedException e) {
            e.printStackTrace();
        }
        return husband;
    }
}

public class Test {
    public static void main(String[] args) {

        Husband h = new Husband();
        Husband h1 = (Husband) h.clone();
        System.out.println("H:\t" + h.birthday + ";" + h.wife);
        System.out.println("H:\t" + h1.birthday + ";" + h1.wife);
        // 浅克隆通常只是对克隆的实例进行复制，但里面的其他子对象，都是共用的
        System.out.println("是否是同一个对象：" + (h == h1)); // false
        System.out.println("是否是同一个引用对象：" + (h.wife == h1.wife)); // true
    }
}
```
#### 深复制（深克隆）
被复制对象的所有的变量都含有与原来的对象相同的值，出去那些引用其他对象的变量。那些引用其他对象的变量将指向被复制过的新对象，而不再是原有的那些被引用的对象。换言之，深复制把要复制的对象所引用的对象都复制了一遍，而这种对被引用到的对象的复制叫做间接复制。
示例：
```
class Wife implements Serializable {
    String name = "范冰冰";
    String birthday = "20151115";

    @Override
    public String toString() {
        return "Wife [name=" + name + ", birthday=" + birthday + "]";
    }
}

class Husband implements Cloneable, Serializable {
    Wife wife = new Wife();
    String birthday = "20151115";

    // 利用串行化深克隆一个对象，把对象以及它的引用读到流里，在写入其他的对象
    public Object deepClone() throws IOException, ClassNotFoundException {
        // 将对象写到流里
        ByteArrayOutputStream bos = new ByteArrayOutputStream();
        ObjectOutputStream oos = new ObjectOutputStream(bos);
        oos.writeObject(this);
        // 从流里读回来
        ByteArrayInputStream bis = new ByteArrayInputStream(bos.toByteArray());
        ObjectInputStream ois = new ObjectInputStream(bis);
        return ois.readObject();
    }
}

public class Test {
    public static void main(String[] args) throws ClassNotFoundException, IOException {

        Husband h = new Husband();
        Husband h1 = (Husband) h.deepClone();
        System.out.println("H:\t" + h.birthday + ";" + h.wife);
        System.out.println("H:\t" + h1.birthday + ";" + h1.wife);
        // 深克隆克隆的时候会复制它的子对象的引用，里面所有的变量和子对象都是又额外拷贝了一份。
        System.out.println("是否是同一个对象：" + (h == h1)); // false
        System.out.println("是否是同一个引用对象：" + (h.wife == h1.wife)); // false
    }
}
```
### 总结
Java语言的构建模型直接支持原始模型模式。所有的JavaBean都继承来自java.lang.Object，而Object类提供一个clone()方法，可以将一个javaBean对象复制一份。但是，这个JavaBean必须实现一个标识接口（Cloneable）,表明这个JavaBean支持复制。
克隆需要满足的条件：
1、对任何的对象x, 都有x.clone() != x。换言之，克隆对象与原对象不是同一个对象；
2、对任何的对象x, 都有x.clone() .getclass() == x.getclass()。换言之，克隆对象与原对象的类型一样；
3、如果对象x的equals()方法定义恰当的话，那么x.clone().equals(x)应当成立的； 

注：以上代码为了简洁，编写的不是太规范忘谅解。