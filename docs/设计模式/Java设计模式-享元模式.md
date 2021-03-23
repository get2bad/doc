# 享元模式

> 什么叫享元模式？我们拆开这两个字:
>
> 享: 共享的意思
>
> 元: 学过JVM肯定听说过元空间吧，其实元空间就相当于内存了
>
> 这就是我们本节课学的享元模式



## 抽象图

![](http://image.tinx.top/img20210323111053.png)





## 作用及解释

> **享元模式**主要用于共享通用对象，减少内存的使用，提升系统访问效率，而这部分共享对象通常比较耗费内存或者需要查询大量接口或者使用数据库资源，因此统一抽离作为共享对象使用.
>
> 另外享元模式可以分为在服务端和客户端：
>
> 一般互联网的H5和Web场景下大部分数据都需要服务端进行处理，比如数据库连接池的使用、多线程线程池的使用，除了这些功能外，还有些需要服务端进行包装后的处理下发给客户端，因为服务端需要做享元处理。
>
> 但在一些游戏场景下，很多都是客户端进行渲染地图效果，比如：树木 花草 鱼虫，通过设置不同元素描述使用享元公用对象，减少内存的占用，让客户端更加流畅



## 应用场景

享元模式经典的应用场景就是池技术了：

+ String常量池
+ 数据库连接池
+ 缓冲池
+ 等等等等



## 情景模拟 - 展示网站项目需求

如果你 / 你的老板接到一个小型的外包项目，给客户A做一个产品展示的网站，客户A的朋友感觉网站不错，也希望做一个这种网站，但是要求有一些不同：

1. 客户B 要求以新闻方式发布新产品
2. 客户C 要求以博客的形式发布新产品
3. 客户D 要求以微信公众号的形式发布新的产品



### 传统方式

#### 传统方式解决思路

1. 直接复制粘贴一份，然后根据客户的不同需求，进行定制修改
2. 给每个网站租用一个空间
3. 方案设计图

![](http://image.tinx.top/img20210323120003.png)

#### 传统方式面临的问题

1. 需要的网站结构相似度很高，而且都不是高访问量的网站，如果分成多个虚拟空间来处理，相当于一个相同网站的实例对象很多，造成服务器的资源浪费造成服务器的资源浪费
2. 解决思路： 整合到一个网站中，共享其相关的代码数据，对于硬盘、内存、CPU、数据库空间等服务器资源都可以达成共享，减少服务器资源
3. 对于代码来说，由于是一份实例，维护和扩展都比较容易

上述情况可以使用享元模式来解决

### 享元模式解决

#### UML图

![](http://image.tinx.top/img20210323133448.png)

+ FlyWeright 是抽象的享元角色，他是产品的抽象类，同时定义出对象的外部状态和内部状态接口或者实现
+ ConcreteFlyWeight 具体享元的角色，是具体的产品类，试下你抽象角色定义相关业务
+ UnSharedConcreteFlyWeight 是不可共享的角色，一般不会出现在享元工厂
+ FlyWeightFactory 享元工厂类，用于构建一个池容器，同时提供池中获取对象方法



#### 解决代码

+ User

```java
@Data
@AllArgsConstructor
@NoArgsConstructor
public class User {

    private String name;
}
```

+ Website

```java
public abstract class Website {

    public abstract void use(User user);
}
```

+ ConcreteWebSite

```Java
public class ConcreteWebSite extends Website {
    private String type = "";

    public ConcreteWebSite(String type) {
        this.type = type;
    }

    @Override
    public void use(User user) {
        System.out.println("网站的发布形式为:" + type + "使用者是：" + user.getName());
    }
}
```

+ WebSiteFactory

```Java
public class WebSiteFactory {

    private Map<String, Website> pool = new HashMap<>();

    public Website getWebsiteCategory(String type){
        if(!pool.containsKey(type)){
            pool.put(type, new ConcreteWebSite(type));
        }
        return pool.get(type);
    }

    public int getWebCount(){
        return pool.size();
    }
}
```

+ Test

```Java
public class Test {

    @org.junit.Test
    public void test(){
        WebSiteFactory factory = new WebSiteFactory();

        Website website1 = factory.getWebsiteCategory("新闻");
        website1.use(new User("隔壁老王"));

        Website website2 = factory.getWebsiteCategory("博客");
        website2.use(new User("隔壁老阮"));

        Website website3 = factory.getWebsiteCategory("微信公众号");
        website3.use(new User("隔壁老baby"));

        Website website4 = factory.getWebsiteCategory("论坛");
        website4.use(new User("隔壁老帅"));
    }
}
```

+ 测试结果

```console
网站的发布形式为:新闻使用者是：隔壁老王
网站的发布形式为:博客使用者是：隔壁老阮
网站的发布形式为:微信公众号使用者是：隔壁老baby
网站的发布形式为:论坛使用者是：隔壁老帅
```



<font color=red>这就是享元模式，说白了，这个设计模式就是告诉我们，对于重复使用的，我们可以使用这个思想来弄一个 池 或者使用合适的中间件(比如Redis)提高对象重用</font>



## JDK中的享元模式 - Integer

+ java.lang.Integer.valueOf

![](http://image.tinx.top/img20210323150201.png)

看途中箭头指的地方，就是利用了缓存，如果创建的数字在 - 128 ~ 127 之间只会返回 缓存中的内容，而不是创建一个对象，这 就是 **享元模式**！！是不是很好玩呢~



好了，今天的设计模式到这里就结束了，您学废了嘛？^_^