# 装饰器模式

装饰器？就是那种将一个东西打扮的漂漂亮亮那种机器嘛？在我们编码过程中是十分常见的，下面跟我见识一下装饰器的魅力吧~



## 抽象图

![](http://image.tinx.top/img20210317104707.png)

> 初看上图感觉装饰器模式有点像俄罗斯套娃 ，⽽装饰器的核⼼就是再不改原有类的基础上 **给类新增功能**。
>
> <font color=red>不改变原有类，可能有的⼩伙伴会想到继承、AOP切⾯，当然这些⽅式都可以实现，但是**使⽤装饰器模式会是另外⼀种思路更为灵活，可以避免继承导致的⼦类过多，也可以避免AOP带来的复杂性**。</font>

## 相关框架中使用装饰器模式的例子

### Spring

Spring 中用到的装饰器模式在类名上有两种表现:一种是类名中含有 Wrapper，另一种是类名中含有Decorator

### IO流

```new BufferedReader(new FileReader(""));``` 我们在学习Java SE时学到IO流的时候相信大家肯定见过这种代码吧！这种代码的作用就是将字节流转换为字符流，而这就是使用装饰器模式的一种体现.



## UML图

![](https://images2017.cnblogs.com/blog/1240966/201710/1240966-20171001210207840-1662607269.png)

+ 抽象构件角色（Component）：通常是一个抽象类或者一个接口，定义了一系列方法，方法的实现可以由子类实现或者自己实现。（例如，对于动物类，有一个抽象方法输出所有的功能，基本功能包括：呼吸，觅食，睡觉等等）
+ 具体构件角色（Concrete Component）：是Component的子类，实现了对应的方法，它就是那个被装饰的类。
+ 装饰角色（Decorator）：是Component的子类，它是具体装饰角色共同实现的抽象类（也可以是接口），并且持有一个Component类型的对象引用，它的主要作用就是把客户端的调用委派到被装饰类。
+ 具体装饰角色（Concrete Decorator）：它是具体的装饰类，是Decorator的子类，当然也是Component的子类。



## 模拟情境

在系统开发初期，我们内部的ERP使用可能仅仅是账户密码判断就行了，但是可能这个企业的发展迅猛，团队中出现了各司其职的角色，每个角色使用ERP的需求不同，有些需要创建活动，有些就只是查看数据，同时为了保证数据的安全性，不会让每个用户都拥有最高的权限。

那么以往使用的SSO单点登录，假设不能在里面添加需要的用户访问验证功能，这时候我们可以使用装饰器模式，扩充原有的单点登录服务，同时也要保证原有功能不收破坏(**假设没有使用 shiro  / Spring Security**)



## 通用类

+ 模拟 Spring 的拦截器

```java
public interface Interceptor {

    boolean preHandle(String req,String resp,Object handler);
}
```

+ 实现功能SSO模拟的登陆拦截器

```java
public class SSOInterceptor implements Interceptor{
    @Override
    public boolean preHandle(String req, String resp, Object handler) {
        // 模拟拿到cookie / header
        String auth = req.substring(1, 8);
        return "success".equals(auth);
    }
}
```



## 没有使用装饰器模式代码实现

+ 重写处理方法

```java
public class NewSSOInterceptor extends SSOInterceptor {

    private static Map<String,String> authMap = new ConcurrentHashMap<>();

    static {
        // 一般都是数据库获取，现在模拟就不通过数据库，尽量简单为主
        authMap.put("wang","admin");
        authMap.put("shuai","admin");
        authMap.put("lao","admin");
    }


    @Override
    public boolean preHandle(String req, String resp, Object handler) {
        // 模拟获取cookie
        String auth = req.substring(1, 8);
        // 模拟校验
        boolean success = auth.equals("success");

        if (!success) return false;
        String userName = req.substring(9);
        String method = authMap.get(userName);
        // 模拟⽅法校验
        return "admin".equals(method);
    }
}
```

+ 测试类 Test

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test(){
        Interceptor interceptor = new NewSSOInterceptor();
        String header = "success:wang";
        log.info("登陆结果：{}",interceptor.preHandle(header,"login","x"));
    }
}
```

+ 结果

```console
11:29:34.178 [main] INFO  com.wills.java.decorate.simple.Test - 登陆结果：true
```



## 装饰器模式重构代码

> + 抽象构件角色（Component）：通常是一个抽象类或者一个接口，定义了一系列方法，方法的实现可以由子类实现或者自己实现。（例如，对于动物类，有一个抽象方法输出所有的功能，基本功能包括：呼吸，觅食，睡觉等等）
> + 具体构件角色（Concrete Component）：是Component的子类，实现了对应的方法，它就是那个被装饰的类。
> + 装饰角色（Decorator）：是Component的子类，它是具体装饰角色共同实现的抽象类（也可以是接口），并且持有一个Component类型的对象引用，它的主要作用就是把客户端的调用委派到被装饰类。
> + 具体装饰角色（Concrete Decorator）：它是具体的装饰类，是Decorator的子类，当然也是Component的子类。

### 装饰器模式模型结构

![](http://image.tinx.top/img20210317113257.png)

+ 抽象类装饰⻆⾊

```java
public abstract class SSODecorator implements Interceptor {

    private Interceptor interceptor;

    public SSODecorator() {
    }

    public SSODecorator(Interceptor interceptor) {
        this.interceptor = interceptor;
    }

    @Override
    public boolean preHandle(String req, String resp, Object handler) {
        System.out.println("其他登陆前的操作");
        return interceptor.preHandle(req,resp,handler);
    }
}
```

+ 装饰⻆⾊逻辑实现

```java
@Slf4j
public class LoginSSODecorator extends SSODecorator{

    private static Map<String,String> authMap = new ConcurrentHashMap<>();

    static {
        // 一般都是数据库获取，现在模拟就不通过数据库，尽量简单为主
        authMap.put("wang","admin");
        authMap.put("shuai","admin");
        authMap.put("lao","admin");
    }

    public LoginSSODecorator() {
    }

    public LoginSSODecorator(Interceptor interceptor) {
        super(interceptor);
    }

    @Override
    public boolean preHandle(String req, String resp, Object handler) {
        operateOther();
        // 模拟获取cookie
        String auth = req.substring(0, 7);
        // 模拟校验
        boolean success = auth.equals("success");

        if (!success) return false;
        String userName = req.substring(8);
        String method = authMap.get(userName);
        // 模拟⽅法校验
        return "admin".equals(method);
    }
    public boolean operateOther(){
        System.out.println("正在处理其他内容");
        return 1 == 1;
    }
}
```

+ 在具体的装饰类实现中，继承了装饰类 SsoDecorator ，那么现在就可以扩展⽅法； preHandle

+ 在 preHandle 的实现中可以看到，这⾥只关⼼扩展部分的功能，同时不会影响原有类的核⼼服务，也不会因为使⽤继承⽅式⽽导致的多余⼦类，增加了整体的灵活性。



### 测试 

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test(){
        LoginSSODecorator interceptor = new LoginSSODecorator(new SSOInterceptor());
        interceptor.operateOther();
        String header = "success:wang";
        log.info("登陆结果：{}",interceptor.preHandle(header,"login","x"));
    }
}
```

结果：

```java
正在处理其他内容
12:03:34.587 [main] INFO  com.wills.java.decorate.upgrade.Test - 登陆结果：true
```



可能以上代码无法理解```LoginSSODecorator interceptor = new LoginSSODecorator(new SSOInterceptor());```

那么我们来说说具体的流程吧：

1. 首先先声明一个拦截器interface
2. 然后使用几个类再实现这个拦截器

3. 声明一个装饰器抽象类，继承第一步的拦截器Interface，实现方法，构造方法有上面第一步讲述的 interface，用于嵌套
4. 创建一个子类 继承 装饰器抽象类，重写Interface中的方法，然后实现自己的方法，构造方法同装饰器抽象类
5. 测试， 创建对象 ```LoginSSODecorator interceptor = new LoginSSODecorator(new SSOInterceptor());```
6. 创建的对量可以灵活调用重写的方法

**这就是装饰器模式，不光有Interface的方法，还有自己的实现，这样我们就可以一层一层的嵌套装饰，然后调用这些一层一层嵌套类的方法**



## 总结

### 优缺点

**优点：**装饰类和被装饰类可以独立发展，不会相互耦合，装饰模式是继承的一个替代模式，装饰模式可以动态扩展一个实现类的功能。

**缺点：**多层装饰比较复杂。

**使用场景：** 1、扩展一个类的功能。 2、动态增加功能，动态撤销。

### 应用举例

**应用实例：**

1. 孙悟空有 72 变，当他变成"庙宇"后，他的根本还是一只猴子，但是他又有了庙宇的功能。 
2. 不论一幅画有没有画框都可以挂在墙上，但是通常都是有画框的，并且实际上是画框被挂在墙上。在挂在墙上之前，画可以被蒙上玻璃，装到框子里；这时画、玻璃和画框形成了一个物体。

