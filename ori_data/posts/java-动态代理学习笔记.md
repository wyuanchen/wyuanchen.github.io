---
title: java-动态代理学习笔记
date: 2017-01-28 10:17:36
categories: Java
tags: Java
---
**代理模式**
    给某个对象提供一个代理对象，并由代理对象控制对于原对象的访问，即客户不直接操控原对象，而是通过代理对象间接地操控原对象。
**其中代理可以分为两种方式,分别是静态代理和动态代理**   

<!-- more -->

## 静态代理
![](http://oke2lzov9.bkt.clouddn.com/blog/images/3_01)

大概的思想就是如果我想创建一个对RealSubject类进行代理的代理类，那么我可以创建一个代理类Proxy，让它实现和RealSubject同样的接口或者同样的函数，也就是实现Subject接口或者继承Subject，这样Proxy也就可以被当做为Subject类来使用，然后让该Proxy类拥有一个RealSubject类的实例，在Proxy类的request()方法中再去调用RealSubject实例的request()方法和做一些其他的处理。  

直接上代码吧

```java
public class ProxyDemo {
    public static void main(String args[]){
        RealSubject subject = new RealSubject();
        Proxy p = new Proxy(subject);
        p.request();
    }
}

interface Subject{
    void request();
}

class RealSubject implements Subject{
    public void request(){
        System.out.println("request");
    }
}

class Proxy implements Subject{
    private Subject subject;
    public Proxy(Subject subject){
        this.subject = subject;
    }
    public void request(){
        System.out.println("PreProcess");
        subject.request();
        System.out.println("PostProcess");
    }
}
```

## 动态代理
如果大量使用前面的静态代理可能就会有人想抱怨了，“我靠，每次实现一个代理我都得去写多一个类”，而且再考虑下以下这种场景：

> 假如我有3个类 AC,BC,CC 分别实现了(换成继承关系也没关系)A,B,C接口， 而且分别实现了fa(),fb(),fc()函数，然后我想通过代理来实现计算fa(),fb(),fc()的执行时间，在这里就简单认为在执行函数前加多个fp()函数吧，如果我是用静态代理的话，那么就意味着我需要分别写多3个代理类，分别为AP,BP,CP类，而且都各自实现A,B,C接口，然后还得在各自代理的函数中加入**同一句**函数fp()，可见这样实现多么死板，这时候我们就需要动态代理来搞定这问题啦!

**首先说下几个词的概念先**

 - 委托类和委托对象：委托类是一个类，委托对象是委托类的实例。
 - 代理类和代理对象：代理类是一个类，代理对象是代理类的实例。

**java实现动态代理有两种:**

 - JDK动态代理
 - cglib动态代理


## JDK动态代理


考虑这么一个例子，假如我有下面这些类
```java
public interface HelloService {
    void sayHello();
}
```

```java
public class HelloServiceImpl implements HelloService {
    public void sayHello(){
        System.out.println("hello world!");
    }
}
```

```java
public class Main {
    public static void main(String[] args){
        HelloService helloService=new HelloServiceImpl();
        helloService.sayHello();
    }
}
```
首先执行结果应该为
```
hello world!
```

 这时候我想通过代理才实现在输出`hello world!`之前先输出`welcome yuan!`,然后在输出`hello world!`之后再输出`bye!`,这时候我需要这么写:
 

```java
public class MyProxyFactory implements InvocationHandler{
    
    //委托对象
    private Object target;

    //构造函数，在此传入将要被代理的对象
    public MyProxyFactory(Object target){
        this.target=target;
    }

    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        //在执行委托对象的函数之前
        System.out.println("welcom yuan!");

        //执行委托对象的函数(target是委托对象，args是这个函数需要的形参)
        Object result=method.invoke(target,args);

        //在执行委托对象的函数之后
        System.out.println("bye!");
        return result;
    }


    //自己封装返回一个代理对象实例
    public Object getProxyInstance(){
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),target.getClass().getInterfaces(),this);
    }

}
```
然后main函数改一下
```java
public class Main {
    public static void main(String[] args){
        HelloService helloService=new HelloServiceImpl();
        MyProxyFactory myProxyFactory=new MyProxyFactory(helloService);
        HelloService helloService1Proxy= (HelloService) myProxyFactory.getProxyInstance();
        helloService1Proxy.sayHello();
    }
}
```
输出结果为

```
welcom yuan!
hello world!
bye!
```
现在解释下关于JDK动态代理用到的类

 1. `java.lang.reflect.Proxy`:这是生成代理类的主类，通过 Proxy 类生成的代理类都继承了 Proxy 类。我们主要通过`Proxy`的
 2. `java.lang.reflect.InvocationHandler`:需要自己定义一个实现该接口的类，我们动态生成的代理类需要完成的具体操作将在该接口的`invoke`方法内实现。 第一个参数是代理对象（表示哪个代理对象调用了method方法），第二个参数是 Method 对象（表示哪个方法被调用了），第三个参数是指定调用方法的参数。这个函数是在代理对象调用任何一个方法时都会调用的，方法不同会导致第二个参数method不同。如果你想对特定的函数进行处理，可以自己在`invoke`方法内自己做些判断，或许还有其他的方法可以实现...

**JDK动态代理的局限性**

> 因为 Java 的单继承特性（每个代理类都继承了 Proxy 类），只能针对接口创建代理类，不能针对类创建代理类。

**附加资料**
[JDK动态代理的实现原理和模拟](http://blog.csdn.net/liushuijinger/article/details/37829049)
[Java 动态代理](http://www.jianshu.com/p/6f6bb2f0ece9)


## cglib动态代理

> 使用JDK动态代理使用到一个Proxy类和一个InvocationHandler接口。
Proxy已经设计得非常优美，但是还是有一点点小小的遗憾之处，那就是它仅支持interface代理（也就是代理类必须实现接口），因为它的设计注定了这个遗憾。对于上面说到JDK仅支持对实现接口的委托类进行代理的缺陷，这个问题CGLIB给予了很好的补位，解决了这个问题，使其委托类也可是非接口实现类。
CGLIB内部使用到ASM，所以下面的例子需要引入asm-3.3.jar、cglib-2.2.2.jar

```java
public class Hello {
    public void sayHello(){
        System.out.println("hello");
    }

    public void sayHi(){
        System.out.println("hi");
    }
}
```

```java
public class MyCglibProxyFactory implements MethodInterceptor{

    //需要实现的接口函数
    //注意！这里的o对象跟被代理对象实例不是同一个
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {


        //过滤那些由Object类继承过来的方法
        //boolean flag=!(method.getDeclaringClass().getName().equals("java.lang.Object"));

        //或者检查要拦截的方法的名字，如果是sayHello方法才进行拦截
        boolean flag=method.getName().equals("sayHello");
        if(flag==true){
            System.out.println("welcome yuan!");
        }

        //我们一般使用proxy.invokeSuper(obj,args)方法。这个很好理解，就是执行原始类的方法。还有一个方法proxy.invoke(obj,args)，这是执行生成子类的方法。
      //如果传入的obj就是子类的话，会发生内存溢出，因为子类的方法不挺地进入intercept方法，而这个方法又去调用子类的方法，两个方法直接循环调用了。
        Object result=methodProxy.invokeSuper(o,args);

        if(flag==true){
            System.out.println("bybe");
        }

        return result;
    }

    /**
     * 返回代理对象实例
     * @param target 被代理的对象实例
     * @return
     */
    public Object getProxyInstance(Object target){
        //该类用于生成代理对象
        Enhancer enhancer=new Enhancer();
        //设置父类
        enhancer.setSuperclass(target.getClass());
        //设置回调用对象为本身
        enhancer.setCallback(this);
        return enhancer.create();
    }
}
```

```java
public class Main {

    public static void main(String[] args){
        Hello hello=new Hello();
        MyCglibProxyFactory myCglibProxyFactory=new MyCglibProxyFactory();
        Hello helloProxy=(Hello)myCglibProxyFactory.getProxyInstance(hello);
        helloProxy.sayHello();
        helloProxy.sayHi();
    }
}
```
**输出结果:**

```
welcome yuan!
hello
bybe
hi
```
其中`sayHello()`方法得到处理，而`sayHi()`方法则没有。

**附加资料**
[cglib参考资料](http://blog.csdn.net/catoop/article/details/50730530)

**[JDK动态代理和CGLib的比较](http://www.kancloud.cn/evankaka/springlearning/119667)**
> CGLib所创建的动态代理对象的性能比JDK所创建的代理对象性能高不少，大概10倍，但CGLib在创建代理对象时所花费的时间却比JDK动态代理多大概8倍，所以对于singleton的代理对象或者具有实例池的代理，因为无需频繁的创建新的实例，所以比较适合CGLib动态代理技术，反之则适用于JDK动态代理技术。另外，由于CGLib采用动态创建子类的方式生成代理对象，所以不能对目标类中的final，private等方法进行处理。所以，大家需要根据实际的情况选择使用什么样的代理了。同样的，Spring的AOP编程中相关的ProxyFactory代理工厂内部就是使用JDK动态代理或CGLib动态代理的，通过动态代理，将增强（advice)应用到目标类中。


    

