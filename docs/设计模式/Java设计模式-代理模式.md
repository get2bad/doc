# 代理模式

> 为一个对象提供一个替身，以控制对这个对象的访问。即通过代理对象访问目标对。这样做的好处就是：<font color=red>可以在目标对象实现的基础上，增强额外的功能操作，即扩展目标对象的功能</font>

## 抽象图

![](http://image.tinx.top/img20210324113842.png)



## 代理方式

1. 静态代理、动态代理(JDK代理、接口代理)
2. Cglib代理(可以在内存动态的创建对象，而不需要实现接口，动态代理的范畴)



## 静态代理

> 静态代理在使用时，需要定义接口或者父类，被代理对象(即目标对象)与代理对象一起实现相同的接口或者继承相同父类

### 小场景 - 商店卖货

+ 商店售卖接口

```java
public interface ISaler {

    public void sale();
}
```

+ 商店售卖实现类

```java
@Slf4j
public class Saler implements ISaler{

    private String prodName;
    private BigDecimal price;

    public Saler(String prodName, BigDecimal price) {
        this.prodName = prodName;
        this.price = price;
    }

    @Override
    public void sale() {
      log.info("正在销售{}商品,价格为：{}~~~",prodName,price);
    }
}
```

+ 售卖代理类

```java
@Slf4j
public class SalerProxy implements ISaler {

    private ISaler saler;

    public SalerProxy(ISaler saler) {
        this.saler = saler;
    }

    @Override
    public void sale() {
        log.info("开始代理~！");
        saler.sale();
        log.info("代理完毕~！");
    }
}
```

+ 测试类

```java
public class Test {

    @org.junit.Test
    public void test(){
        Saler saler = new Saler("隔壁老王的爱心包裹❤️",new BigDecimal(4.00));
        new SalerProxy(saler).sale();
    }
}
```

+ 测试结果

```console
12:09:53.575 [main] INFO  c.w.j.p.u.static_proxy.SalerProxy - 开始代理~！
12:09:53.579 [main] INFO  com.wills.java.proxy.common.Saler - 正在销售隔壁老王的爱心包裹❤️商品,价格为：4~~~
12:09:53.579 [main] INFO  c.w.j.p.u.static_proxy.SalerProxy - 代理完毕~！
```

### 静态代理优缺点

#### 优点

+ 在不修改目标对象的功能前提下，能通过代理对象对目标功能扩展

#### 缺点

+ 因为代理对象需要与目标对象实现一样的接口，所以会有很多代理类
+ 一旦接口增加方法，目标对象和代理对象都要维护



## 动态代理

> 1. 代理对象不需要实现接口，但是目标对象需要实现接口，否则不能用动态代理
> 2. 代理对象的生成，是利用JDK的API，动态的在内存中构建代理对象
> 3. 动态代理也叫做：JDK代理，接口代理

### 动态代理小例子 - 商店卖货

+ ISaler（同上）
+ Saler(同上)
+ ProxyFactory

```java
@Slf4j
public class ProxyFactory {

    private Object target;

    public ProxyFactory(Object target) {
        this.target = target;
    }

    public Object getProxyInstance(){
        return Proxy.newProxyInstance(target.getClass().getClassLoader(),
                target.getClass().getInterfaces(),
                new InvocationHandler() {
            @Override
            public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
                log.info("JDK开始代理卖货咯~");
                // 执行原本类的方法
                Object invoke = method.invoke(target, args);
                log.info("JDK结束代理卖货咯~");
                return invoke;
            }
        });
    }
}
```

+ 测试类Test

```java
public class Test {

    @org.junit.Test
    public void test(){
        ISaler saler = new Saler("隔壁老王的爱心包裹❤️",new BigDecimal(4.00));
        ISaler proxy = (ISaler)new ProxyFactory(saler).getProxyInstance();
        proxy.sale();
    }
}
```

+ 测试结果

```console
13:17:02.154 [main] INFO  c.w.j.p.u.dynamic_proxy.ProxyFactory - JDK开始代理卖货咯~
13:17:02.158 [main] INFO  com.wills.java.proxy.common.Saler - 正在销售隔壁老王的爱心包裹❤️商品,价格为：4~~~
13:17:02.159 [main] INFO  c.w.j.p.u.dynamic_proxy.ProxyFactory - JDK结束代理卖货咯~
```



## Cglib 进行代理

> 静态代理和JDK大力模式都是要求目标对象实现一个接口，但是有时候目标对象只是一个单独的对象，并没有实现任何接口，这个时候可以使用目标对象子类来实现代理 - cglib
>
> Cglib代理也叫做子类代理，它可以在运行期间扩展JAVA类与实现java接口，它被广泛的应用在大部分的AOP框架中，例如<font color=red>Spring AOP</font>
>
> AOP编程中如何选择代理模式：
>
> 1. 如果目标类实现了接口，那么使用JDK代理
>
> 2. 目标对象不需要实现接口，那么使用CGlib代理
>
> **Cglib包的底层是通过使用字节码处理框架ASM来转换字节码并生成新类**



### CGLib 的 maven依赖

```xml
<dependency>
    <groupId>cglib</groupId>
    <artifactId>cglib</artifactId>
    <version>2.2.2</version>
</dependency>
```



### cglib代理小例子 - 商店卖货

+ 类中用到的对象同上
+ ProxyFactory

```java
@Slf4j
public class ProxyFactory implements MethodInterceptor {

    private Object target;

    public ProxyFactory(Object target) {
        this.target = target;
    }

    public Object getProxyInstance(){
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(target.getClass());
        enhancer.setCallback(this);
        return enhancer.create();
    }

    @Override
    public Object intercept(Object o, Method method, Object[] args, MethodProxy methodProxy) throws Throwable {
        log.info("JDK开始代理卖货咯~");
        // 执行原本类的方法
        Object invoke = method.invoke(target, args);
        log.info("JDK结束代理卖货咯~");
        return invoke;
    }
}
```

+ Test

```java
public class Test {

    @org.junit.Test
    public void test(){
        Saler saler = new Saler("隔壁老王的爱心包裹❤️",new BigDecimal(4.00));
        Saler proxy = (Saler) new ProxyFactory(saler).getProxyInstance();
        proxy.sale();
    }
}
```

+ 测试结果

```console
13:33:32.970 [main] INFO  c.w.j.p.u.cglib_proxy.ProxyFactory - JDK开始代理卖货咯~
13:33:32.973 [main] INFO  com.wills.java.proxy.common.Saler - 正在销售隔壁老王的爱心包裹❤️商品,价格为：4~~~
13:33:32.974 [main] INFO  c.w.j.p.u.cglib_proxy.ProxyFactory - JDK结束代理卖货咯~
```



## 总结

在日常开发的我们，可以合理的去使用 各种设计模式来让我们的代码更加有效 且 智能，看完代理模式的朋友们，你们有没有跃跃欲试想自己做点什么呢？还不快去试试~！



今天，您学废了嘛？

