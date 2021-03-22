# 外观模式

> 朋友们，如果你没有听说过外观模式，看到这个名字大概就能知晓其意思了，没错！这就是集万众宠爱的外观模式，他是 蔡徐坤割割、肖战割割，哈哈哈哈，开个玩笑，这个外观模式的本意呢，就是 **看得到的外观**，他是啥意思呢，容我慢慢道来~



## 抽象图

![](http://image.tinx.top/img20210322110049.png)



## 模式含义

<font color=red>外观模式也叫⻔⾯模式，主要解决的是降低调⽤⽅的使⽤接⼝的复杂逻辑组合。</font>

<font color=red>这样调⽤⽅与实际的接⼝提供⽅提供⽅提供了⼀个中间层，⽤于包装逻辑提供API接⼝。有些时候外观模式也被⽤在中间件层，对服务中的通⽤性复杂逻辑进⾏中间件层包装，让使⽤⽅可以只关⼼业务开发。</font>



## 具体场景

如果我们很早接触互联网的话，大家一定会有这个经历，之前社交都是由QQ / 论坛为主，每一次注册，都会填写很多的基本信息，就像：

姓名、手机号、昵称、邮箱、性别、家庭住址、是否单身等等等等

经过时代的发展，现在我们注册一个网站大部分就是扫个码 或者 输入手机号即可，仅仅一步就可以注册成功。站在程序开发的角度呢，就是以前服务提供商提供了一整套的接口，供我们进行注册调用，而现在仅仅是提供一个接口我们就可以注册成功，这就是外观模式。



## 演示小案例 - 影院管理项目

### 场景导入

组建一个家庭影院: DVD播放器、投影仪、环绕立体声，要求完成使用家庭影院功能，其过程为：

直接用遥控器：统筹各设备开关

放下屏幕

开投影仪

开音响

开DVD

选DVD

调暗灯光

播放

观影结束，关闭各种设备

调亮灯光



### 通用代码

#### 设备

```java
public interface Equipment {

    public void turnOn();

    public void turnOff();
}
```

#### 灯

```java
public class Light implements Equipment{

    private static Light light = new Light();

    private Light(){}

    public static Light getInstance(){
        return light;
    }

    @Override
    public void turnOn(){
        System.out.println("开灯");
    }

    public void turnDarker(){
        System.out.println("调暗灯光");
    }

    @Override
    public void turnOff(){
        System.out.println("关灯");
    }
}
```

#### 投影仪

```java
public class Projector implements Equipment{

    // 使用饿汉式单例模式
    private static Projector projector = new Projector();

    private Projector(){}

    public static Projector getInstance(){
        return projector;
    }

    @Override
    public void turnOn(){
        System.out.println("开启投影仪");
    }

    @Override
    public void turnOff(){
        System.out.println("关闭投影仪");
    }
}
```

#### 屏幕

```java
public class Screen implements Equipment{

    private static Screen screen = new Screen();

    private Screen(){}

    public static Screen getInstance(){
        return screen;
    }

    @Override
    public void turnOn(){
        System.out.println("放下投影仪");
    }

    @Override
    public void turnOff(){
        System.out.println("拉起投影仪");
    }
}
```

#### 音响

```java
public class Speaker implements Equipment{

    private static Speaker speaker = new Speaker();

    private Speaker(){}

    public static Speaker getInstance(){
        return speaker;
    }

    @Override
    public void turnOn(){
        System.out.println("打开音响");
    }

    @Override
    public void turnOff(){
        System.out.println("关闭音响");
    }
}
```

#### DVD播放器

```java
public class DVDPlayer implements Equipment{

    private static DVDPlayer dvdPlayer = new DVDPlayer();

    private DVDPlayer(){}

    public static DVDPlayer getInstance(){
        return dvdPlayer;
    }

    @Override
    public void turnOn(){
        System.out.println("打开DVD播放机");
    }

    public void choseDVD(){
        System.out.println("挑选DVD");
    }

    public void play(){
        System.out.println("播放DVD");
    }

    @Override
    public void turnOff(){
        System.out.println("关闭DVD播放机");
    }
}
```



### 没有使用外观模式的代码

```java
public class Test {

    @org.junit.Test
    public void test(){
        Screen screen = Screen.getInstance();
        Light light = Light.getInstance();
        Projector projector = Projector.getInstance();
        Speaker speaker = Speaker.getInstance();
        DVDPlayer dvdPlayer = DVDPlayer.getInstance();
        System.out.println("-----------------观影-----------------");
        screen.turnOn();
        projector.turnOn();
        speaker.turnOn();
        dvdPlayer.turnOn();
        dvdPlayer.choseDVD();
        light.turnDarker();
        dvdPlayer.play();
        dvdPlayer.turnOff();
        speaker.turnOff();
        projector.turnOff();
        screen.turnOff();
        light.turnLighter();
        System.out.println("--------------------------------------");

    }
}
```

运行结果：

```console
-----------------观影-----------------
放下投影仪
开启投影仪
打开音响
打开DVD播放机
挑选DVD
调暗灯光
播放DVD
关闭DVD播放机
关闭音响
关闭投影仪
拉起投影仪
调亮灯光
--------------------------------------
```

大家看到上述代码是不是感觉太繁琐，糟糕了，每次都要写这么多的吗？如果有一小点马虎，那么就可能导致业务逻辑不正确，这要怎么办？



### 使用外观模式

#### 抽象图

![](http://image.tinx.top/img20210322120812.png)

#### 外观类

> 就是对一套业务逻辑的封装，使用户直接调用一个方法，就可以使用一套复杂的业务流程

```java
@Slf4j
public class HomePlayFace {

    private DVDPlayer dvdPlayer;
    private Light light;
    private Projector projector;
    private Screen screen;
    private Speaker speaker;

    public HomePlayFace() {
        dvdPlayer = DVDPlayer.getInstance();
        light = Light.getInstance();
        projector = Projector.getInstance();
        screen = Screen.getInstance();
        speaker = Speaker.getInstance();
    }

    public void ready(){
        log.info("---------------设备准备-------------------");
        screen.turnOn();
        projector.turnOn();
        speaker.turnOn();
        dvdPlayer.turnOn();
        dvdPlayer.choseDVD();
        light.turnDarker();
        log.info("----------------------------------------");
    }

    public void play(){
        log.info("---------------开始放映-------------------");
        dvdPlayer.play();
        log.info("----------------------------------------");
    }

    public void stop(){
        log.info("---------------结束放映-------------------");
        dvdPlayer.turnOff();
        speaker.turnOff();
        projector.turnOff();
        screen.turnOff();
        light.turnLighter();
        log.info("----------------------------------------");
    }
}
```

#### 测试

```java
public class Test {

    @org.junit.Test
    public void test(){
        HomePlayFace face = new HomePlayFace();
        face.ready();
        face.play();
        face.stop();
    }
}
```

#### 运行结果

```console
12:07:28.551 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ---------------设备准备-------------------
放下投影仪
开启投影仪
打开音响
打开DVD播放机
挑选DVD
调暗灯光
12:07:28.556 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ----------------------------------------
12:07:28.556 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ---------------开始放映-------------------
播放DVD
12:07:28.557 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ----------------------------------------
12:07:28.557 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ---------------结束放映-------------------
关闭DVD播放机
关闭音响
关闭投影仪
拉起投影仪
调亮灯光
12:07:28.557 [main] INFO  c.w.j.a.upgrade.HomePlayFace - ----------------------------------------
```



## 相关框架中的外观模式

### Mybatis

mybatis框架中有外观模式哦，没想到吧，我们看一下 ```org.apache.ibatis.session.Configuration```类的newMetaObject方法

![](http://image.tinx.top/img20210322121130.png)

我们追踪进去看一下：

![](http://image.tinx.top/img20210322130708.png)

我们再追踪，查看这个类的带参构造方法

![](http://image.tinx.top/img20210322130841.png)

类图

![](http://image.tinx.top/img20210322131015.png)

## 总结

1. 外观模式对外屏蔽了子系统的细节，因此外观模式降低了客户端对子系统的使用复杂性
2. 外观模式对客户端子系统的耦合关系 - 解耦，让子系统内部的模块变得更易维护和扩展
3. 通过合理的使用外观模式，可以帮我们更好的**划分访问的层级**
4. 当系统需要进行分层设计时，可以考虑使用Facade模式
5. **在维护一个遗留的大型系统时，如果这个系统变得极其难以维护和扩展，此时可以考虑为新系统开发一个Facade类，来提供遗留系统比较清晰简单的接口，让新系统与Facade类交互，减少不必要的麻烦**