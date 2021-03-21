# 单例模式

本节没啥好说的，就是使用这个模式可以保证我们运行的程序内的获取某一个对象是唯一的，我在第二节使用过饿汉式单例模式，有兴趣的可以去翻看一下~



## 单例模式抽象图

![](http://image.tinx.top/img20210313140323.png)



单例模式在我们项目中或者框架中应该很常见，比如说

+ spring中的bean，默认创建就是singleton模式(单例)，确保项目中的bean都是同一个
+ 数据库连接池不会反复创建
+ 代码中全局变量



下面跟着我的代码，来学习一下常见的单例模式吧~

### 静态类

> 这种静态类的方式就是充分利用了 java中 static关键字的特性.
>
> static关键字的作用：
>
> 静态变量是随着类加载时被完成初始化的，它在内存中仅有一个，且JVM也只会为它分配一次内存，同时类所有的实例都共享静态变量，可以直接通过类名来访问它

```java
public class StaticSingleton {

    public static StaticSingleton data = new StaticSingleton();

    private StaticSingleton(){}
}
```

### 懒汉模式 - 线程不安全

> 使用的时候才创建对象，但是面临一个问题，就是 **线程不安全** 因为多个线程进来访问 getInstance方法的时候都会进行
> if(null != instance) 这行代码，如果同一时间的instance都是null 那么都会直接创建对象

```java
public class LazySingletonByThreadless {

    private static LazySingletonByThreadless singleton;

    // 私有构造方法
    private LazySingletonByThreadless(){}

    public static LazySingletonByThreadless getInstance(){
        if(null != singleton) return singleton;
        singleton = new LazySingletonByThreadless();
        return singleton;
    }
}
```

### 懒汉模式 - **线程安全**

> 啥？线程不安全？加锁！ synchronized 喂饱饱！

```java
public class LazySingletonByThread {

    private static LazySingletonByThread singleton;

    private LazySingletonByThread(){}

    public static synchronized LazySingletonByThread getInstance(){
        if(null != singleton) return singleton;
        singleton = new LazySingletonByThread();
        return singleton;
    }
}
```

### 饿汉式 - **线程安全**

> 什么是 饿汉式？ 就是创建即是对象，创建了就立马创建对象，因为立即创建了，所以线程安全

```java
public class HungrySingletonByThread {

    private static HungrySingletonByThread singleton = new HungrySingletonByThread();

    private HungrySingletonByThread(){}

    public static HungrySingletonByThread getInstance(){
        return singleton;
    }
}
```

###  使⽤静态内部类(**线程安全**)

> 什么是内部类，就是类里面再包裹一层类，这在 java图形界面 java swing很常见

```java
public class InnerStaticClass {

    private InnerStaticClass(){}

    private static class Instance{
        private static InnerStaticClass instance = new InnerStaticClass();
    }

    public static InnerStaticClass getInstance(){
        return Instance.instance;
    }
}
```

### 双重锁校验

> 就是将方法级的锁，替换为了对象级锁，减少了部分获取实例的耗时

```java
public class ObjectLockSingleton {

    private static ObjectLockSingleton singleton;

    private ObjectLockSingleton(){}

    public static ObjectLockSingleton getInstance(){
        if(null != singleton) return singleton;
        synchronized (ObjectLockSingleton.class){
            singleton = new ObjectLockSingleton();
            return singleton;
        }
    }
}
```

### CAS锁方式单例模式 - 线程安全

> 我们使用 CAS 锁的方式来实现线程安全

```java
public class CASSingleton {

    private static final AtomicReference<CASSingleton> INSTANCE = new AtomicReference<>();

    private CASSingleton(){}

    public static final CASSingleton getInstance(){
        for (;;){
            CASSingleton singleton = INSTANCE.get();
            if(null != singleton) return singleton;
            INSTANCE.compareAndSet(null,new CASSingleton());
            return INSTANCE.get();
        }
    }
}
```

### Effect java作者推荐的枚举方式单例模式

> 枚举默认就是独一份的，所以直接使用枚举就可以了

```java
public enum EnumSingleton {

    INSTANCE;

    private static TestObject instance = new TestObject();

    public TestObject getInstance(){
        return instance;
    }
}
```





测试一下~

### 测试

> 由于为了展示效果，我们不使用 junit提供的 Assert 单元测试，因为那个效果不明显直观

1. 静态类

```java
@org.junit.Test
public void testStaticSingleton(){
    System.out.println(StaticSingleton.data);
    System.out.println(StaticSingleton.data);
}
```

```java
// 测试通过，打印的地址一样！
com.wills.java.singleton.upgrade.StaticSingleton@22927a81
com.wills.java.singleton.upgrade.StaticSingleton@22927a81
```

2. 懒汉式 - 线程不安全

```java
@org.junit.Test
public void testLazySingletonByThreadless(){
    System.out.println(LazySingletonByThreadless.getInstance());
    System.out.println(LazySingletonByThreadless.getInstance());
}
```

```java
com.wills.java.singleton.upgrade.LazySingletonByThreadless@22927a81
com.wills.java.singleton.upgrade.LazySingletonByThreadless@22927a81
```

3. 懒汉式 - 线程安全

```java
@org.junit.Test
public void testLazySingletonByThread(){
    System.out.println(LazySingletonByThread.getInstance());
    System.out.println(LazySingletonByThread.getInstance());
}
```

```java
com.wills.java.singleton.upgrade.LazySingletonByThreadless@22927a81
com.wills.java.singleton.upgrade.LazySingletonByThreadless@22927a81
```

4. 饿汉式 - 线程安全

```java
@org.junit.Test
public void testHungrySingletonByThread(){
    System.out.println(HungrySingletonByThread.getInstance());
    System.out.println(HungrySingletonByThread.getInstance());
}
```

```java
com.wills.java.singleton.upgrade.HungrySingletonByThread@22927a81
com.wills.java.singleton.upgrade.HungrySingletonByThread@22927a81
```

5. 静态内部类

```java
@org.junit.Test
public void testInnerStaticClass(){
    System.out.println(InnerStaticClass.getInstance());
    System.out.println(InnerStaticClass.getInstance());
}
```

```java
com.wills.java.singleton.upgrade.InnerStaticClass@22927a81
com.wills.java.singleton.upgrade.InnerStaticClass@22927a81
```

6. 双重校验

```java
@org.junit.Test
public void testObjectLockSingleton(){
    System.out.println(ObjectLockSingleton.getInstance());
    System.out.println(ObjectLockSingleton.getInstance());
}
```

```console
com.wills.java.singleton.upgrade.ObjectLockSingleton@22927a81
com.wills.java.singleton.upgrade.ObjectLockSingleton@22927a81
```

7. CAS锁单例模式

```java
@org.junit.Test
public void testCASSingleton(){
    System.out.println(CASSingleton.getInstance());
    System.out.println(CASSingleton.getInstance());
}
```

```console
com.wills.java.singleton.upgrade.CASSingleton@78e03bb5
com.wills.java.singleton.upgrade.CASSingleton@78e03bb5
```

8. 枚举类测试

```java
@org.junit.Test
public void testEnum(){
    System.out.println(EnumSingleton.INSTANCE.getInstance());
    System.out.println(EnumSingleton.INSTANCE.getInstance());
}
```

```console
com.wills.java.singleton.common.TestObject@22927a81
com.wills.java.singleton.common.TestObject@22927a81
```



通过以上的测试，我们拿到的所有对象都是独一份的，如果您在开发中遇到了需要全局一个变量，何不从这8种选择挑选一份自己喜欢的呢~~~



好了今天的单例模式到这里就结束了，我们下一节就开始讲 ```结构性设计模式```，精彩继续，不要错过哦~

