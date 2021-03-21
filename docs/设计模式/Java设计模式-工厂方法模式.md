# Java设计模式-工厂方法模式

![](http://image.tinx.top/img20210309103025.png)

> 工厂模式又称工厂方法模式，是一种创建型设计模式，其在父类中提供一个创建对象的方法，允许子类决定实例化对象的类型
>
> 
>
> 这个工厂模式的主要目的是定义一个创建对象的接口，让自己决定实例化哪一个工厂类，工厂模式使其创建过程延迟到子类进行

首先呢，我们先上一段模拟的线上糟糕的代码：

```java
public CommonResult sendAward(AwardReq awardReq) throws InterruptedException {
    log.info("开发发放奖品：{}", awardReq.toString());
    Thread.sleep(2000);
    String res = "";
    // 开始发放奖品
    if(awardReq.getAwardType() == 1){
        // 正常的Springboot项目自动注入，因为这个是基础的maven项目，暂时使用创建对象
        CouponService service = new CouponService();
        // 发放奖品
        res = service.sendCoupon(awardReq.getUserId(), awardReq.getAwardNumber());
        log.info("发放奖品结果:{}",res);
    } else if(awardReq.getAwardType() == 2){
        GoodsService goodsService = new GoodsService();
        res = goodsService.sendGoods(awardReq.getUserId(), awardReq.getAwardNumber());
        log.info("发放商品结果:{}",res);
    } else if(awardReq.getAwardType() == 3){
        VIPService vipService = new VIPService();
        res = vipService.sendGoods(awardReq.getUserId(),awardReq.getAwardNumber());
        log.info("发放vip结果:{}",res);
    }
    return CommonResult.ok(res);
}
```

看到这一坨代码有没有很熟悉？除了Service使用自动注入以外，别的和我们平常写的代码几乎一样，但是我们需要思考一下：

1. 如果加了一个类型的，要咋办？再加一层 else if ??? 那岂不是这一坨代码还要再往下加，这样项目迭代到后期，就会变得极其难易维护，首先要弄懂每一项awardType代表的含义(如果不注释的话，现在一大堆人不喜欢注释，后期维护变得和屎山一样的恶心)，还得进去看看其运作流程，这代价有点大了
2. 代码过长，后期如果需求变动你要怎么弄？



所以，有一个规范的写代码流程变得极其重要了，所以抽象方法工厂设计模式帮你解决问题：

抽象工厂方法如何解决上述的问题呢？请看下图(不规范的UML画图，后期会进行改正)：

![](http://image.tinx.top/img20210309125909.png)

因为UML图不太规范，那么我们直接上代码来解决大家的困惑吧：

1. 工厂抽象接口 + 方法

```java
public interface Factory {
    String sendAward(AwardReq req) throws Exception;
}

```

2. 工厂接口实现类，然后实现具体方法
   1. 奖券工厂

```java
public class CouponFactory implements Factory{


    @Override
    public String sendAward(AwardReq req) throws Exception {
        if("".equals(req.getAwardNumber())){
            return "发放奖品失败，奖品序列码不正确！";
        } else {
            return "发送奖品成功！";
        }
    }
}
```

​		2. 商品工厂

```java
public class GoodsFactory implements Factory{
    @Override
    public String sendAward(AwardReq req) throws Exception {
        if("".equals(req.getAwardNumber())){
            return "发放商品失败，奖品序列码不正确！";
        } else {
            return "发送商品成功！";
        }
    }
}
```

​		3. VIP发放工厂

```java
public class VIPFactory implements Factory{
    @Override
    public String sendAward(AwardReq req) throws Exception {
        if("".equals(req.getAwardNumber())){
            return "发放VIP失败，奖品序列码不正确！";
        } else {
            return "发放VIP成功！";
        }
    }
}
```

3. 工厂生产类

```java
public class FactoryStore {

    public Factory getFactoryService(Integer awardType){
        if(awardType == null) return null;
        if(awardType == 1) return new CouponFactory();
        if(awardType == 2) return new GoodsFactory();
        if(awardType == 3) return new VIPFactory();
        throw new RuntimeException("不存在的奖品类型！");
    }
}
```

4. 工厂测试

```java
@Slf4j
public class FactoryMethodMode {
    @Test
    public void test() throws Exception {
        FactoryStore store = new FactoryStore();
        AwardReq req = new AwardReq("1","qskf24FJDsidg3aiFF","asdfhdj3DJHFD",1,"abccccc","aaaa");
        // 测试 其中这个req.getAwardType()的不同，工厂生产类会生产给我们不同的实现对象
        log.info("测试发放奖券结果：{}",store.getFactoryService(req.getAwardType()).sendAward(req));
    }
}
```

结合我的代码，再去看图，是不是就很清晰了呢？

这样在业务逻辑层就不需要写上过多的逻辑代码，直接让工厂生产类生产我们所需要的工厂实现类即可，然后调用其方法，即可完成业务



思考：

1. 这么写是在业务层面上节省了很多代码，那么可维护性怎么样？

   论可维护性，如果我们增加了新的发放奖品的方式，那么我们只需创建一个新的类，来实现工厂Factory，然后在工厂生产类加入其生产逻辑即可，这样，我们业务层代码就可以真正做到了0改动，是不是很省心呢？

2. 如果添加了新的业务需求咋办？

   如果所有的方式都要用到这个需求，那么就可以在Factory抽象接口上添加新需求的方法，然后在其实现类中实现新加需求的代码即可，这样做的话结构化更佳。

   如果有某些方式不需要实现这些需求，再如果工厂内部管理着100+的实现类，我们每个都要去实现一次，岂不是累死我了？

   没事！我们可以再进一步，使用抽象工厂模式，就是使用一个抽象abstract工厂来实现，因为抽象类可以有选择的使用重写，如果不重写，那么就直接调用通用的抽象类的方法内容(第二节 抽象工厂模式会详细给大家讲解！！！！)



以上就是抽象方法模式的全部内容，下面我来总结一下：

> 工厂模式的意义：
>
> 将实例化对象的代码提取出来，放到一个类中统一管理和维护，达到和主项目的依赖关系的解耦，从而提高项目的扩展考和维护性

使用工厂方法的优点：

1. 避免创建者与具体的产品逻辑耦合
2. 满⾜单⼀职责，每⼀个业务逻辑实现都在所属⾃⼰的类中完成
3. 满⾜开闭原则，⽆需更改使⽤调⽤⽅就可以在程序中引⼊新的产品类型

缺点：

使用工厂方法模式也会带来⼀些问题，⽐如有⾮常多的奖品类型，那么实现的⼦类会极速扩张。因此也需要使⽤其他的模式进⾏优化，这些在后续的设计模式中会逐步涉及到