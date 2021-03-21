# 抽象工厂模式

**前言：**

不知道大家有没有遇到这种情况： 前离职程序员**“码品”**极差，写代码一点都不注重规范，造出来很多屎山代码，恶心至极，自己硬着头皮上，然后因为项目催的急，只能屎上加屎的感觉？(例如：拼音变量，拼音方法名，无限嵌套if else，三重for循环，代码分层就是个摆设，写的代码东一块西一块，这种行为我直接在上面加注释 挂出来git提交人姓名 + 提交时间 后面猛批 狗屎代码，兄弟能改就改，不能改抓进跑路，出事可是要背锅的！！！)，这种东西最终会导致工程代码臃肿不堪，线上频出事故，代码重构极其困难，最终离职走人。

通过以上的吐槽，让我们知道有一手好的**码品**很重要，这样，设计模式就是我们的救命稻草，这就是许多**大厂必须要求会设计模式的原因**。



**抽象工厂模式图：**

![](http://image.tinx.top/img20210310104102.png)



本案例使用 Redis 双集群升级作为小例子讲解抽象工厂模式

![](http://image.tinx.top/img20210310104947.png)

在这里我举个小例子：

>  如果公司开发一个客户端，但是每一类系统对应的SDK不同，这样导致我们的应用在书写时就变得极其困难，简单的解决办法就是使用if else，不过后面添加了新的系统支持，都要全部变动那一坨一坨的if else? 这肯定是不可能的吧，这也太累人了，用一下工厂模式，几分钟搞定，剩余时间上班摸摸鱼不舒服么~~~~

在这里，我拿Redis切换来举个例子：

在项目刚启动的初期，肯定因为有某些业务用到了Redis，就匆匆的安装了一台单机redis服务，但是呢，由于项目前景很棒，导致系统的QPS / 并发访问压力过大，这样我们就不得不考虑使用Redis集群，一是为了更好的应对以上的问题，二是为了更好地 容灾 准备。但是从单机 切换至 集群，如果做了容灾备份的话，那就是 单机 → 集群A → 集群B ，但是呢，每个集群的api不太相同，下面看我的代码，我来模拟一下每个集群不同的api方法

**为了图方便，我们使用模拟的Redis机器，使用饿汉式单例工厂模式创建对象(第5节会降到单例工厂模式)**

+ Redis模拟类

```java
public class Redis {
    // 私有化构造方法
    private Redis(){}

    // 线程安全的 ConcurrentHashMap
    private final static Map<Object,Object> data = new ConcurrentHashMap<>();

    public static Map<Object,Object> getInstance(){
        return data;
    }
}
```

+ 单机

```java
@Slf4j
public class SingleRedis {

    public String get(String key){
        log.info("Redis获取数据{}",key);
        return Redis.getInstance().get(key);
    }

    public void set(String key,String value){
        log.info("Redis设置值{}:{}",key,value);
        Redis.getInstance().put(key,value);
    }

    public void set(String key, String value, long timeExpire, TimeUnit timeUnit){
        log.info("Redis设置值{}:{},过期时间为:{},时间单位：{}",key,value,timeExpire,timeUnit.toString());
        Redis.getInstance().put(key,value);
    }

    public void del(String key){
        log.warn("Redis删除值：{}",key);
        Redis.getInstance().remove(key);
    }
}
```

+ 集群A

```java
@Slf4j
public class ClusterA {

    public String gain(String key){
        log.info("Redis获取数据{}",key);
        return Redis.getInstance().get(key);
    }

    public void set(String key,String value){
        log.info("Redis设置值{}:{}",key,value);
        Redis.getInstance().put(key,value);
    }

    public void setEx(String key, String value, long timeExpire, TimeUnit timeUnit){
        log.info("Redis设置值{}:{},过期时间为:{},时间单位：{}",key,value,timeExpire,timeUnit.toString());
        Redis.getInstance().put(key,value);
    }

    public void delete(String key){
        log.warn("Redis删除值：{}",key);
        Redis.getInstance().remove(key);
    }
}
```

+ 集群B

```java
@Slf4j
public class ClusterB {

    public String get(String key){
        log.info("Redis获取数据{}",key);
        return Redis.getInstance().get(key);
    }

    public void set(String key,String value){
        log.info("Redis设置值{}:{}",key,value);
        Redis.getInstance().put(key,value);
    }

    public void setExpire(String key, String value, long timeExpire, TimeUnit timeUnit){
        log.info("Redis设置值{}:{},过期时间为:{},时间单位：{}",key,value,timeExpire,timeUnit.toString());
        Redis.getInstance().put(key,value);
    }

    public void del(String key){
        log.warn("Redis删除值：{}",key);
        Redis.getInstance().remove(key);
    }
}
```

调用代码的接口类

```java
public interface CacheService {

    String get(String key);

    void set(String key,String value);

    void set(String key, String value, long timeExpire, TimeUnit timeUnit);

    void del(String key);
}

```

通过上述的代码大家可以看出来，每个集群的相关的api方法不尽相同，为了统一，我们定义了一个接口类，然后根据不同的方式书写实现代码



下面，我来模拟一下我们没有使用设计模式的代码：

+ CacheServiceImpl

```java
public class CacheServiceImpl implements CacheService {

    private Integer type;

    private Map<String, String> data;

    private SingleRedis singleRedis;
    private ClusterA clusterA;
    private ClusterB clusterB;

    public CacheServiceImpl() {
    }

    public CacheServiceImpl(Integer type) {
        this.type = type;
        singleRedis = new SingleRedis();
        clusterA = new ClusterA();
        clusterB = new ClusterB();
        data = Redis.getInstance();
    }

    @Override
    public String get(String key) {
        if(type != null){
            if(1 == type){
                return singleRedis.get(key);
            } else if(2 == type){
                return clusterA.gain(key);
            } else if(3 == type){
                return clusterB.get(key);
            } else {
                return null;
            }
        }
        return null;
    }

    @Override
    public void set(String key, String value) {
        if(type != null){
            if(1 == type){
                singleRedis.set(key,value);
            } else if(2 == type){
                clusterA.set(key,value);
            } else if(3 == type){
                clusterB.set(key, value);
            }
        }
    }

    @Override
    public void set(String key, String value, long timeExpire, TimeUnit timeUnit) {
        if(type != null){
            if(1 == type){
                singleRedis.set(key,value,timeExpire,timeUnit);
            } else if(2 == type){
                clusterA.setEx(key,value,timeExpire,timeUnit);
            } else if(3 == type){
                clusterB.setExpire(key,value,timeExpire,timeUnit);
            }
        }
    }

    @Override
    public void del(String key) {
        if(type != null){
            if(1 == type){
                singleRedis.del(key);
            } else if(2 == type){
                clusterA.delete(key);
            } else if(3 == type){
                clusterB.del(key);
            }
        }
    }
}
```



下面我们来测试一下这种代码：

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test1(){
        CacheService service1 = new CacheServiceImpl(1);
        service1.set("author","王小懒☁️");

        CacheService service2 = new CacheServiceImpl(2);
        log.info(service1.get("author"));

        CacheService service3 = new CacheServiceImpl(3);
        service3.del("author");
    }
}
```

测试结果：

```console
11:40:42.593 [main] INFO com.wills.java.factory.factory_interface.common.SingleRedis - Redis设置值author:王小懒☁️
11:40:42.598 [main] INFO com.wills.java.factory.factory_interface.common.SingleRedis - Redis获取数据author
11:40:42.598 [main] INFO com.wills.java.factory.factory_interface.simple.Test - 王小懒☁️
11:40:42.598 [main] WARN com.wills.java.factory.factory_interface.common.ClusterB - Redis删除值：author
```

感觉这样的代码，可以不可以呢？假如说我还得加集群，加的集群的方法还是有某些不同呢？ 难道每个方法都要 ```if else```? 肯定不现实吧！



这样，我们来使用抽象工厂模式来拯救这种的代码

我们对上面进行重构，重构图如下所示：

![](http://image.tinx.top/img20210310140358.png)

重构设计的类功能作用：

+ ICacheAdapter: 定义了适配接口，分别包装两个集群中差异化的接口名称。 ClusterAAdapter / ClusterBAdapter

+ JDKProxy / JDKInvocationHandler, 是代理类的定义和实现，这部分也就是抽象工厂的另外一种实现方式。通过这样的方式可以很好的把原有的操作Redis方法进行代理操作，通过控制不同的传入对象，控制缓存的作用(**对于Java代理、反射不熟练的童鞋记得加强一下自己这方面的知识积累**)



下面，show your code~:

+ ICacheAdapter

  > 这个类的主要作⽤是让所有集群的提供⽅，能在统⼀的⽅法名称下进⾏操作。也⽅⾯后续的拓展。

```java
public interface ICacheAdapter {

    String get(String key);

    void set(String key, String value);

    void set(String key, String value, long timeout, TimeUnit timeUnit);

    void del(String key);
}
```

+ ClusterAAdapter

```java
public class ClusterAAdapter implements ICacheAdapter{

    ClusterA clusterA;

    public ClusterAAdapter() {
         clusterA = new ClusterA();
    }

    @Override
    public String get(String key) {
        return clusterA.gain(key);
    }

    @Override
    public void set(String key, String value) {
        clusterA.set(key, value);
    }

    @Override
    public void set(String key, String value, long timeout, TimeUnit timeUnit) {
        clusterA.setEx(key, value, timeout, timeUnit);
    }

    @Override
    public void del(String key) {
        clusterA.delete(key);
    }
}
```

+ ClusterBAdapter

```java
public class ClusterBAdapter implements ICacheAdapter{

    ClusterB clusterB;

    public ClusterBAdapter() {
        clusterB = new ClusterB();
    }

    @Override
    public String get(String key) {
        return clusterB.get(key);
    }

    @Override
    public void set(String key, String value) {
        clusterB.set(key, value);
    }

    @Override
    public void set(String key, String value, long timeout, TimeUnit timeUnit) {
        clusterB.setExpire(key, value, timeout, timeUnit);
    }

    @Override
    public void del(String key) {
        clusterB.del(key);
    }
}
```

下面是重点，拿出小本本记下来！！！

+ JDKProxy

```java
public class JDKProxy {

    public static <T> T getProxy(Class<T> interfaceClass, ICacheAdapter cacheAdapter) throws Exception {
        InvocationHandler handler = new JDKInvocationHandler(cacheAdapter);
        ClassLoader classLoader = Thread.currentThread().getContextClassLoader();
        Class<?>[] classes = interfaceClass.getInterfaces();
        return (T) Proxy.newProxyInstance(classLoader, new Class[]{classes[0]}, handler);
    }
}
```

+ JDKInvocationHandler

```java
public class JDKInvocationHandler implements InvocationHandler {

    private ICacheAdapter iCacheAdapter;

    public JDKInvocationHandler(ICacheAdapter iCacheAdapter) {
        this.iCacheAdapter = iCacheAdapter;
    }


    @Override
    public Object invoke(Object proxy, Method method, Object[] args) throws Throwable {
        return ICacheAdapter.class.getMethod(method.getName(), ClassLoaderUtils.getClazzByArgs(args)).invoke(iCacheAdapter,args);
    }

    static class ClassLoaderUtils{
        public static Class<?>[] getClazzByArgs(Object[] args) {
            Class<?>[] parameterTypes = new Class[args.length];
            for (int i = 0; i < args.length; i++) {
                if (args[i] instanceof ArrayList) {
                    parameterTypes[i] = List.class;
                    continue;
                }
                if (args[i] instanceof LinkedList) {
                    parameterTypes[i] = List.class;
                    continue;
                }
                if (args[i] instanceof HashMap) {
                    parameterTypes[i] = Map.class;
                    continue;
                }
                if (args[i] instanceof Long){
                    parameterTypes[i] = long.class;
                    continue;
                }
                if (args[i] instanceof Double){
                    parameterTypes[i] = double.class;
                    continue;
                }
                if (args[i] instanceof TimeUnit){
                    parameterTypes[i] = TimeUnit.class;
                    continue;
                }
                parameterTypes[i] = args[i].getClass();
            }
            return parameterTypes;
        }
    }
}
```



下面是激动人心的小测试了~

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test1() throws Exception{
        CacheService proxyClusterA = JDKProxy.getProxy(CacheServiceImpl.class,new ClusterAAdapter());
        proxyClusterA.set("author","王小懒☁️");

        CacheService proxyClusterB = JDKProxy.getProxy(CacheServiceImpl.class,new ClusterBAdapter());
        log.info(proxyClusterB.get("author"));
    }
}
```

结果

```console
13:39:12.587 [main] INFO  c.w.j.f.f.common.ClusterA - Redis设置值author:王小懒☁️
13:39:12.593 [main] INFO  c.w.j.f.f.common.ClusterB - Redis获取数据author
13:39:12.593 [main] INFO  c.w.j.f.f.upgrade.Test - 王小懒☁️
```

这个就是可以无缝对接自适应各种API的抽象工厂类，使用到了JDK代理以及反射，其核心原理就是判定调用的代理对象，然后拿到方法的参数，自动判定调用哪个方法，这样当我们添加新的集群时几乎可以做到业务代码0改动，是不是很好玩~ 快来尝试一下吧！



思考：

每当我们发现自己要写的代码需要有一大坨的if else if else时，我们就可以思考是否可以使用 抽象工厂 / 工厂方法模式来解决我们当前需要做的业务逻辑，涉及到后期的改动多不多，如果这个地方频繁变动需求 / 增加新特性，那么我们今天和昨天讲得工厂模式就派上了极大的用场，真正可以做到业务代码0变动的需求！



总结：

  工厂模式就是将我们想要拿到的目标对象不再以if else + new Object()的方法进行创建，而是使用某种代理类帮助我们进行创建，每当我们变动代码时，可以直接变动代理类，不需要改动真实的业务逻辑代码，即节省时间又看起来井井有条



