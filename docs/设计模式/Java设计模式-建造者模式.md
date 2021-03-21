# 建造者模式

请问大名鼎鼎的lombok用过吗？，我们可以在项目中直接使用@Builder注解来充分体验一下建造者模式的魅力，本节我们讲的是建造者模式，下面跟着我深入建造者模式的内部，深度探究一下它的实现原理吧~



## 建造者模式抽象图



![](http://image.tinx.top/img20210311111133.png)

本图几乎就可以很直接明了的说明了建造者模式就是我们从0对一个目标对象不断组装，最终得到我们想要的"机器人"，组装的机器人按照我们的口味可以不尽相同。



## 真实场景

我们举一个真实场景的案例吧：

​	我之前的公司把我外派到北京TATA木门做Java开发，线上现在有一个需求，就是要推出不同的套餐服务，一般会有：欧式豪华、轻奢田园、现代简约等等，而这些套餐的背后是不同的商品组合。例如：欧式静音门、简约门套、雕花子门、哑光淡雅油漆等等等等，然后按照不同的套餐价格选取不同的品牌组合，最终再按照装修面积给出一个整体的保价。

​	这⾥我们就模拟公司想推出⼀些套餐装修服务，按照不同的价格设定品牌选择组合，以达到使⽤建造者模式的过程。



> 以下代码，除了规范的接口外，其他同类产品都有两个品牌，不同的价格，供我们测试用。 这里的实体类代码可以略过不看，因为没啥看的

+ Main （接口，规定了每个产品的基础信息）

```java
public interface Main {
    // 名称
    String name();
    // 品牌
    String brand();
    // 价格
    BigDecimal price();
    // 描述
    String desc();
}
```

+ ODoor (欧式门实体类)

```java
public class ODoor implements Main{
    @Override
    public String name() {
        return "欧式轻奢静音门";
    }

    @Override
    public String brand() {
        return "隔壁老王";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(3400);
    }

    @Override
    public String desc() {
        return "奢华的欧式风格静音门，让您在静谧的生活中享受不一样的感觉";
    }
}
```

+ ZDoor(中式门实体类)

```java
public class ZDoor implements Main{
    @Override
    public String name() {
        return "中式奢华静音门";
    }

    @Override
    public String brand() {
        return "中华";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(3600);
    }

    @Override
    public String desc() {
        return "中式的奢华们不香么，兄弟们！";
    }
}
```

+ WChildDoor(小王子门实体类)

```java
public class WChildDoor implements Main{
    @Override
    public String name() {
        return "简约子门";
    }

    @Override
    public String brand() {
        return "隔壁小王";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(1200);
    }

    @Override
    public String desc() {
        return "隔壁小王牌，质量顶呱呱";
    }
}
```

+ HChildDoor(嘿嘿门实体类)

```java
public class HChildDoor implements Main{
    @Override
    public String name() {
        return "哈哈子门";
    }

    @Override
    public String brand() {
        return "哈哈";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(333);
    }

    @Override
    public String desc() {
        return "哈哈哈哈哈哈哈哈哈哈哈";
    }
}
```

+ WDoorPocket(王老实门套实体类)

```java
public class WDoorPocket implements Main{
    @Override
    public String name() {
        return "王老实门套";
    }

    @Override
    public String brand() {
        return "王老实";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(800);
    }

    @Override
    public String desc() {
        return "王老实门套，这小门套，嘎嘎好！";
    }
}
```

+ JDoorPocket(接盘侠门套实体类)

```java
public class JDoorPocket implements Main{
    @Override
    public String name() {
        return "接盘侠门套";
    }

    @Override
    public String brand() {
        return "接盘侠";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(666.66);
    }

    @Override
    public String desc() {
        return "我就是接盘侠，我会好好对待他的孩子的！";
    }
}
```

+ YPaint(雅力士油漆实体类)

```java
public class YPaint implements Main{
    @Override
    public String name() {
        return "棕色哑光油漆";
    }

    @Override
    public String brand() {
        return "雅力士";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(88.88);
    }

    @Override
    public String desc() {
        return "好！👌";
    }
}
```

+ DPaint(多乐士油漆)

```java
public class DPaint implements Main{
    @Override
    public String name() {
        return "棕色亮晶晶油漆";
    }

    @Override
    public String brand() {
        return "多乐士";
    }

    @Override
    public BigDecimal price() {
        return new BigDecimal(66.66);
    }

    @Override
    public String desc() {
        return "俺觉得👌！";
    }
}
```



### 不使用设计模式的代码：

```java
public class GoodsController {

    public CommonResult getMainList(Integer level){
        if(level == null) return CommonResult.faild();

        Map<String,Object> res = new HashMap<>();

        List<Main> goodsList = new ArrayList<>();
        BigDecimal price = BigDecimal.ZERO;

        if(level == 1){
            // 欧式轻奢门
            ODoor oDoor = new ODoor();
            JDoorPocket jDoorPocket = new JDoorPocket();
            DPaint dPaint = new DPaint();
            HChildDoor hChildDoor = new HChildDoor();

            goodsList.add(oDoor);
            goodsList.add(jDoorPocket);
            goodsList.add(dPaint);
            goodsList.add(hChildDoor);

            price = price.add(oDoor.price()).add(jDoorPocket.price()).add(dPaint.price()).add(hChildDoor.price());
        }

        if(level == 2){
            // 中式奢华们
            ZDoor zDoor = new ZDoor();
            WDoorPocket wDoorPocket = new WDoorPocket();
            WChildDoor wChildDoor = new WChildDoor();
            YPaint yPaint = new YPaint();

            goodsList.add(zDoor);
            goodsList.add(wDoorPocket);
            goodsList.add(wChildDoor);
            goodsList.add(yPaint);

            price = price.add(zDoor.price()).add(wDoorPocket.price()).add(yPaint.price()).add(wChildDoor.price());
        }

        if(level == 3){
            // 套餐 1
            ODoor oDoor = new ODoor();
            WDoorPocket wDoorPocket = new WDoorPocket();
            YPaint yPaint = new YPaint();
            WChildDoor wChildDoor = new WChildDoor();

            goodsList.add(oDoor);
            goodsList.add(wDoorPocket);
            goodsList.add(wChildDoor);
            goodsList.add(yPaint);

            price = price.add(oDoor.price()).add(wDoorPocket.price()).add(yPaint.price()).add(wChildDoor.price());
        }
        StringBuilder sb = new StringBuilder();
        goodsList.stream().forEach(data -> {
            sb.append("\r\n商品：" + data.brand() + ":" + data.name() + ":" + data.desc());
        });
        res.put("goods",sb);
        res.put("price", price.setScale(2,BigDecimal.ROUND_HALF_UP));
        return CommonResult.ok(res);
    }
}
```

测试：

```java
13:28:03.257 [main] INFO  com.wills.java.builder.simple.Test - CommonResult(res=OK, data={price=4466.32, goods=
商品：隔壁老王:欧式轻奢静音门:奢华的欧式风格静音门，让您在静谧的生活中享受不一样的感觉
商品：接盘侠:接盘侠门套:我就是接盘侠，我会好好对待他的孩子的！
商品：多乐士:棕色亮晶晶油漆:俺觉得👌！
商品：哈哈:哈哈子门:哈哈哈哈哈哈哈哈哈哈哈}, msg=null, reqToken=57220fe97e62472b87c6da5215a89bf1)
```

瞧瞧这恶心的代码，看的真的是想吐，每一个都要判断一次？如果变换了套餐还得加一种if else?这太不现实了



### 使用设计模式的代码

其实使用建造者模式并不难，他的核心类就三个

1. Builder ，建造者类具体的各种组装由此类实现。

```java
public class Builder {

    public IMenu level1(){
        return new DecorationPackageMenu(1)
                .appendDoor(new ODoor())
                .appendChildDoor(new WChildDoor())
                .appendDoorPocket(new JDoorPocket())
                .appendPaint(new YPaint());
    }

    public IMenu level2(){
        return new DecorationPackageMenu(1)
                .appendDoor(new ZDoor())
                .appendChildDoor(new WChildDoor())
                .appendDoorPocket(new WDoorPocket())
                .appendPaint(new YPaint());
    }

    public IMenu level3(){
        return new DecorationPackageMenu(1)
                .appendDoor(new ODoor())
                .appendChildDoor(new HChildDoor())
                .appendDoorPocket(new JDoorPocket())
                .appendPaint(new YPaint());
    }

    public IMenu level4(){
        return new DecorationPackageMenu(1)
                .appendDoor(new ODoor())
                .appendChildDoor(new WChildDoor())
                .appendDoorPocket(new JDoorPocket())
                .appendPaint(new DPaint());
    }
}
```

2. IMenu， 建造的接口类

```java
public interface IMenu {
    // 门
    IMenu appendDoor(Main door);
    // 子门
    IMenu appendChildDoor(Main door);
    // 门套
    IMenu appendDoorPocket(Main door);
    // 油漆
    IMenu appendPaint(Main door);
  
  	String getDetail();
}
```

3. DecorationPackageMenu ，是 IMenu 接⼝的实现类，主要是承载建造过程中的填充器。相当于这是⼀套承载物料和创建者中间衔接的内容。

```java
public class DecorationPackageMenu implements IMenu{

    private List<Main> list = new ArrayList<Main>(); // 套餐清单
    private BigDecimal price = BigDecimal.ZERO; // 套餐价格

    private Integer packetId;

    public DecorationPackageMenu(Integer packetId) {
        this.packetId = packetId;
    }

    @Override
    public IMenu appendDoor(Main door) {
        list.add(door);
        price = price.add(door.price());
        return this;
    }

    @Override
    public IMenu appendChildDoor(Main childDoor) {
        list.add(childDoor);
        price = price.add(childDoor.price());
        return this;
    }

    @Override
    public IMenu appendDoorPocket(Main doorPocket) {
        list.add(doorPocket);
        price = price.add(doorPocket.price());
        return this;
    }

    @Override
    public IMenu appendPaint(Main paint) {
        list.add(paint);
        price = price.add(paint.price());
        return this;
    }

    @Override
    public String getDetail(){
        StringBuilder sb = new StringBuilder();
        sb.append("\r\n----------套餐" + this.packetId +"-------------\r\n");
        sb.append("总价：" + this.price.setScale(2,BigDecimal.ROUND_HALF_UP) + "\r\n");
        for (Main main : list) {
            sb.append("商品：" + main.brand());
            sb.append("名称：" + main.name());
            sb.append("价格：" + main.price().setScale(2,BigDecimal.ROUND_HALF_UP));
            sb.append("介绍：" + main.desc());
        }
        sb.append("\r\n-------------------------------------------------\r\n");
        return sb.toString();
    }
}
```

测试用例：

```java
@Slf4j
public class Test {

    @org.junit.Test
    public void test(){
        Builder builder = new Builder();
        log.info(builder.level1().getDetail());
    }
}
```

输出：

```console
13:52:14.711 [main] INFO  com.wills.java.builder.upgrade.Test - 
----------套餐1-------------
总价：5355.54
商品：隔壁老王名称：欧式轻奢静音门价格：3400.00介绍：奢华的欧式风格静音门，让您在静谧的生活中享受不一样的感觉商品：隔壁小王名称：简约子门价格：1200.00介绍：隔壁小王牌，质量顶呱呱商品：接盘侠名称：接盘侠门套价格：666.66介绍：我就是接盘侠，我会好好对待他的孩子的！商品：雅力士名称：棕色哑光油漆价格：88.88介绍：好！👌
-------------------------------------------------
```



上面使用设计模式的关键点就在```IMenu```接口上，可以自由拼装我们需要的东西，然后一股脑的返回，是不是很好玩呢？



好了，这就是建造者模式，蛮简单的，但是用处挺大的，因为我们在开发的过程中会遇到很多参数不同，目的相同的东西，不妨我们都可以重置成 建造者模式，岂不是节省时间去摸鱼了？/滑稽



总结：

+ 通过上⾯对建造者模式的使⽤，已经可以摸索出⼀点⼼得。那就是什么时候会选择这样的设计模式，当：``` ⼀些基本物料不会变，⽽其组合经常变化的时候``` ，就可以选择这样的设计模式来构建代码。

+ 此设计模式满⾜了单⼀职责原则以及可复⽤的技术、建造者独⽴、易扩展、便于控制细节⻛险。但同时当出现特别多的物料以及很多的组合后，类的不断扩展也会造成难以维护的问题。但这种设计结构模型可以把重复的内容抽象到数据库中，按照需要配置。这样就可以减少代码中⼤量的重复。

+ 设计模式能带给你的是⼀些思想，但在平时的开发中怎么样清晰的提炼出符合此思路的建造模块，是⽐较难的。需要经过⼀些锻炼和不断承接更多的项⽬，从⽽获得这部分经验。有的时候你的代码写的好，往往是倒逼的，复杂的业务频繁的变化，不断的挑战！