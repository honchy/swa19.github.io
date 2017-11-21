---
layout: post
title:  "Java扩展机制-SPI"
date:   2017-11-19 00:45:27 +0800
categories: 基础
tags: java
---

昨天看Dubbo中关于服务注册和管理部分的时候，接触到了java的扩展机制，了解下实现方式，后边自己写框架的时候也能用下

# SPI解决了什么问题

根据昨天对Dubbo代码的跟踪，SPI可以根据配置动态选择实现的子类，并对子类做注入。如果没有这种机制的话需要怎么实现呢？
为了避免调用者了解实现细节，不能将bean注入的事情交给调用者去做，所以可行的方式是调用者做配置，服务提供者根据配置对类做注入。
那么在根据配置做注入的时候，一种是if-else，这种硬编码的方式不利于做扩展，后边每次新增一个实现类的话，都需要修改代码；另一种是把每个子类的都事先做实例化，这样根据配置可以获取到对应的bean，但是如果一个子类的实例化开销比较大，这种方式就变得可行度不高了。
所以，SPI做了什么呢？SPI是ServiceProviderInterfaces的简称，也就是服务提供接口，通过SPI服务，在需要扩增基类的接口的时候，只需要在jar包的资源文件下增加一个指定子类的文件即可，对原有的模块没有任何其他影响。
首先看一下简单的demo
创建一个基类和两个实现类

~~~
package basicExercise.spi;

/**
 * Created by jinyan on 11/19/17 2:49 PM.
 */
public interface Base {
    void print();
}
~~~


~~~
package basicExercise.spi;

/**
 * Created by jinyan on 11/19/17 2:49 PM.
 */
public class BaseImpl1 implements Base {

    @Override
    public void print() {
        System.out.println("base1");
    }
}
~~~

~~~
package basicExercise.spi;

/**
 * Created by jinyan on 11/19/17 2:50 PM.
 */
public class BaseImpl2 implements Base {

    @Override
    public void print() {
        System.out.println("base2");
    }
}
~~~

在资源文件的META-INF/services下创建一个名为basicExercise.spi.Base（基类的FullName），文件内容为

~~~
basicExercise.spi.BaseImpl1
~~~

也就是其中一个子类的FullName
最后创建一个测试类：

~~~
    public static void main(String[] args) {
        ServiceLoader<Base> sl = ServiceLoader.load(Base.class);
        Iterator<Base> s = sl.iterator();
        if (s.hasNext()) {
            Base ss = s.next();
            ss.print();
        }
    }
~~~

运行main方法，控制台打印`base1`,说明print执行的是BaseImpl1中的方法。试想如果需要增加一个BaseImpl3的print方法，那么只需要修改资源文件中的basicExercise.spi.Base中文件内容即可，系统中无需做代码修改，这种扩展方式在工程项目中是非常借鉴的。在Dubbo中，提供了多种注册中心，如果未来增加了一个注册中心，Dubbo代码中只需要引入新的jar包即可，Dubbo的调用中如果需要使用未来的这个注册中心，也只需要以原有的方式配置register即可，这就体现了高扩展性

# jdk 实现原理
以上对SPI的应用使用的是jdk提供的工具类ServiceLoader，这个类有两个约定好的规则
1. 资源文件要放置在“META-INF/services/”文件夹下
2. 资源文件的命名需要按照基类的class全称命名，如示例中的“basicExercise.spi.Base”
3. 资源文件中添加的内容为加载的子类的class全名，且不可换行
在调用Service.load(Class class)方法主要做了两件事情：
1. 初始化ServiceLoader
2. 对遍历器做初始化，在s.hasNext遍历时，加载资源文件：
~~~
        private boolean hasNextService() {
            if (nextName != null) {
                return true;
            }
            if (configs == null) {
                try {
                    String fullName = PREFIX + service.getName();
                    if (loader == null)
                        configs = ClassLoader.getSystemResources(fullName);
                    else
                        configs = loader.getResources(fullName);
                } catch (IOException x) {
                    fail(service, "Error locating configuration files", x);
                }
            }
            while ((pending == null) || !pending.hasNext()) {
                if (!configs.hasMoreElements()) {
                    return false;
                }
                pending = parse(service, configs.nextElement());
            }
            nextName = pending.next();
            return true;
        }
~~~
其中`ClassLoader.getSystemResources(fullName)`会根据拼接好的资源文件名查找资源文件，并通过`nextName = pending.next();`获取到资源文件中设定的子类的名称
接下来，示例代码中的`next()`方法获取到子类的实例
~~~
        private S nextService() {
            if (!hasNextService())
                throw new NoSuchElementException();
            String cn = nextName;
            nextName = null;
            Class<?> c = null;
            try {
                c = Class.forName(cn, false, loader);
            } catch (ClassNotFoundException x) {
                fail(service,
                     "Provider " + cn + " not found");
            }
            if (!service.isAssignableFrom(c)) {
                fail(service,
                     "Provider " + cn  + " not a subtype");
            }
            try {
                S p = service.cast(c.newInstance());
                providers.put(cn, p);
                return p;
            } catch (Throwable x) {
                fail(service,
                     "Provider " + cn + " could not be instantiated",
                     x);
            }
            throw new Error();          // This cannot happen
        }
~~~
`S p = service.cast(c.newInstance())`把原有的基类的实例映射成为子类的实例，最终返回的`Base ss = s.next()`ss对象就成为了设定好的子类的实例。

# Dubbo对SPI的扩展
Dubbo中对jdk的扩展机制做了进一步的扩展，整体的实现位于`com.alibaba.dubbo.common.extension`包下
做加载工作的类为`ExtensionLoader`




