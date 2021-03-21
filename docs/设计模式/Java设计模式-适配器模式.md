# 适配器模式

这个设计模式见名知意，就是应对不同的场合，我们使用适配器来应对不同的环境。

如果大家技术还可以的话，肯定会去尝试去读一下我们日常开发的框架，比如说 SpringBoot / Mybatis / springcloud alibaba nacos 等等等，如果阅读过源码的话，肯定见过蛮多名字后缀为Adapter的类，这些类就是我们本节要讨论的适配器模式的产物，现在对这个适配器模式有了一些概念以后，我们就开始本节课的学习吧~！



## 情境导入

![](https://img2018.cnblogs.com/blog/1272523/201901/1272523-20190129142942342-2089942613.png)

![](https://img2018.cnblogs.com/blog/1272523/201901/1272523-20190129143032772-477362990.png)

大家如果有接触过 PS4 PS5 XBOX等游戏机的都知道，因为某些政策原因，我们大陆地区无法玩到血腥暴力的游戏，但是恰巧那种游戏才是真正的大作，那怎么办？我们就只能去购买 港版机器，这种地区的机器不会存在锁区锁游戏的问题，但是有一个问题就是，港版游戏机采用了外国的电源标准，所以，我们的电源线插头处必须要准备一个电源**适配器**，适配器的作用就是将之前的东西转换为我们现在需要的东西。



## 项目模拟

假设我们的公司发展前景非常好，当基础的系统逐渐成型后，业务运行就需要开始做用户的拉新和促活，从而保障新用户的增加以及最终的现金转换。

熟悉拼多多的都知道，他们的营销之一就是： 你邀请⼀个⽤户开户、或者邀请⼀个⽤户下单，那么平台就会给你返利，多邀多得。同时随着拉新的量越来越多开始设置每⽉下单都会给⾸单奖励，等等，各种营销场景。

**那么这个时候做这样⼀个系统就会接收各种各样的MQ消息或者接⼝，如果⼀个个的去开发，就会耗费很⼤的成本，同时对于后期的拓展也有⼀定的难度。此时就会希望有⼀个系统可以配置⼀下就把外部的MQ接⼊进⾏，这些MQ就像上⾯提到的可能是⼀些注册开户消息、商品下单消息等等**

那么这种场景可以用到适配器模式嘛？答案是可以的！

以下是系统内通用的类，供我们测试使用

+ 用户注册实体类

```java
@Data
public class CreateUser {

    private Integer id;
    private String userName;
    private String pwd;
    private Date createTime;
    private String remark;
    private Integer type;
}
```

+ 订单实体类

```java
@Data
public class CreateUser {

    private Integer id;
    private String userName;
    private String pwd;
    private Date createTime;
    private String remark;
    private Integer type;
}
```



## 通用解决方案

如果我们平常解决多个MQ消息，然后分别处理一般都是有几个业务，我们就建立几个类，然后在其中处理就OK了，下面是示例代码：

```java
@Slf4j
public class CreateMQ {

    public void onMessage(String message){
        CreateUser user = JSON.parseObject(message, CreateUser.class);
        // 处理相关的业务 ...
    }
}
```



但是，考虑一下，我们要处理的MQ业务特别多，有100+咋办？每个都建个类？这不可能吧？？？

所以，这时候 适配器模式就显得特别重要了，那么我们使用适配器模式来优化一下吧~



## 适配器模式

> 适配器模式要解决的主要问题就是多种差异化类型的接⼝做统⼀输出，这在我们学习⼯⼚⽅法模式中也有所提到不同种类的奖品处理，其实那也是适配器的应⽤。

### 适配器模型结构

![](http://image.tinx.top/img20210315115507.png)

+ 统一的MQ消息体

```java
@Data
public class RebateInfo {

    private String userId;
    private String bizId;
    private Date biaTime;
    private String desc;
}
```

+ MQ适配消息适配

> 这个类里面的方法，就是用于把不同的MQ转换成我们要的各种属性，但是为什么方法上面是Map呢？因为我们接收的信息几乎都是json信息，所以我们直接将 json转换为map，这样就不需要适配实体类了，然后用反射将map转换为我们上面写的统一的MQ消息体

```java
public class MQAdapter {

public static RebateInfo filter(String strJson, Map<String, String>
        link) throws Exception{
    return filter(JSON.parseObject(strJson, Map.class), link);
}

public static RebateInfo filter(Map obj, Map<String, String> link)
        throws Exception{
    RebateInfo rebateInfo = new RebateInfo();
    for (String key : link.keySet()) {
        Object val = obj.get(link.get(key));
        RebateInfo.class.getMethod("set" + key.substring(0, 1).toUpperCase() + key.substring(1), String.class)
                .invoke(rebateInfo, val.toString());
    }
    return rebateInfo;
}
```



好了，适配已经oK了，可以进行测试了！

```java
public class Test {

    @org.junit.Test
    public void test() throws Exception {
        CreateUser createUser = new CreateUser();
        createUser.setCreateTime(new Date().toString());
        createUser.setUserName("隔壁老王呀！");
        createUser.setPwd(MessageDigest.getInstance("md5").digest("12345".getBytes()).toString());
        createUser.setType(1);

        // 定义转换的Map
        Map<String,String> link1 = new HashMap<>();
        link1.put("userId","userId");
        link1.put("bizTime","createTime");
        link1.put("name","userName");
        link1.put("type","type");
        link1.put("desc","pwd");

        RebateInfo filter = MQAdapter.filter(JSON.toJSONString(createUser), link1);
        System.out.println(filter);
        System.out.println("---------------------------------------------");
        Order order = new Order();
        order.setOrderId("123456");
        order.setOrderTime(new Date().toString());
        order.setPrice(new BigDecimal(7));
        order.setSku("df23tg4d");
        order.setSkuName("测试");

        // 定义转换的Map
        Map<String,String> link2 = new HashMap<>();
        link2.put("bizId","orderId");
        link2.put("bizTime","String");
        link2.put("name","skuName");
        link2.put("type","sku");
        link2.put("desc","price");

        System.out.println(MQAdapter.filter(JSON.toJSONString(order), link2));
    }
}
```

运行答案：

```console
RebateInfo(userId=null, name=隔壁老王呀！, bizId=null, bizTime=Mon Mar 15 13:27:45 CST 2021, desc=[B@78e03bb5, type=1)
---------------------------------------------
RebateInfo(userId=null, name=测试, bizId=123456, bizTime=Mon Mar 15 13:27:45 CST 2021, desc=7, type=df23tg4d)
```



使用这一个类，就可以直接转换我们接收的信息然后转换成统一的实体类，就可以进行相关的操作了~！



如果是接口的适配器，那么就是写一个接口，提供同一个方法，最后相关方法实现，然后执行即可，差不多就是我们平常开发经常使用的 Interface + implement方式，在这里就不做介绍了。



总结： 

+ 使用适配器模式，可以让我们减少很多不必要的创建类 + 逻辑，减少了大量重复的判断 也使项目更加易于扩展



学习设计模式不一定非要遵循我上面的例子来，但是里面的精髓以及意识一定要在以后的编码前要想一下，是否可以使用相关设计模式让我们的代码更加简单高效，说不定还能省出来**摸鱼**的时间呢？😁



