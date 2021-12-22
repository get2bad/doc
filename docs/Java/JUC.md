[toc]

# Synchornized

> 读音： ['sɪŋkrənaɪzd]
>
>  记得刚刚开始学习Java的时候，一遇到多线程情况就是synchronized，相对于当时的我们来说synchronized是这么的神奇而又强大，那个时候我们赋予它一个名字“同步”，也成为了我们解决多线程情况的百试不爽的良药。但是，随着我们学习的进行我们知道synchronized是一个重量级锁，相对于Lock，它会显得那么笨重，以至于我们认为它不是那么的高效而慢慢摒弃它。 诚然，随着Javs SE 1.6对synchronized进行的各种优化后，synchronized并不会显得那么重了。下面是 Java是如何对它进行了优化、锁优化机制、锁的存储结构和升级过程；

## 实现原理

> synchronized可以保证方法或者代码块在运行时，同一时刻只有一个方法可以进入到临界区，同时它还可以保证共享变量的内存可见性

Java中每一个对象都可以作为锁，这是synchronized实现同步的基础：

1. 普通同步方法，锁是当前实例对象
2. 静态同步方法，锁是当前类的class对象
3. 同步方法块，锁是括号里面的对象

当一个线程访问同步代码块时，它首先是需要得到锁才能执行同步代码，当退出或者抛出异常时必须要释放锁，那么它是如何来实现这个机制的呢？我们先看一段简单的代码：

```java
public class ThreadTest {

    public void hello(){
        System.out.println(123);
    }

    public void say(){
        synchronized (this){
            System.out.println(456);
        }
    }
}
```

然后我们使用``` javac ThreadTest.java```来编译一下这个类，然后使用```javap -c XXX/ThreadTest.class```来看一下class文件内的```Synchornized```的实现:

![](http://image.tinx.top/20211119141757.png)我们可以从上图中看出来，我们声明的say方法中有加入synchornized关键字，在于上面的类查看时，发现多出了两个``` monitorenter``` 和 ```monitorexit``` 关键字，经过查询，这两个关键字就是synchornized字节码解释！

> - **同步代码块**：monitorenter指令插入到同步代码块的开始位置，monitorexit指令插入到同步代码块的结束位置，JVM需要保证每一个monitorenter都有一个monitorexit与之相对应。任何对象都有一个monitor与之相关联，当且一个monitor被持有之后，他将处于锁定状态。线程执行到monitorenter指令时，将会尝试获取对象所对应的monitor所有权，即尝试获取对象的锁；
> - **同步方法**：synchronized方法则会被翻译成普通的方法调用和返回指令如:invokevirtual、areturn指令，在VM字节码层面并没有任何特别的指令来实现被synchronized修饰的方法，而是在Class文件的方法表中将该方法的access_flags字段中的synchronized标志位置1，表示该方法是同步方法并使用调用该方法的对象或该方法所属的Class在JVM的内部对象表示Klass做为锁对象。

## Java对象头的组成

+ Mark Word (标记字段)
  + 用于存储对象自身的运行时数据，它是实现轻量级锁和偏向锁的关键
+ Klass Pointer(类型指针)
  + Klass Point是是对象指向它的类元数据的指针，虚拟机通过这个指针来确定这个对象是哪个类的实例

### Mark Word

MarkWord用于存储对象自身的运行时数据，如哈希码（HashCode）、GC分代年龄、锁状态标志、线程持有的锁、偏向线程 ID、偏向时间戳等等。Java对象头一般占有两个机器码（在32位虚拟机中，1个机器码等于4字节，也就是32bit），但是如果对象是数组类型，则需要三个机器码，因为JVM虚拟机可以通过Java对象的元数据信息确定Java对象的大小，但是无法从数组的元数据来确认数组的大小，所以用一块来记录数组长度。下图是Java对象头的存储结构（32位虚拟机）

| 25bit          | 4bit           | 1bit       | 2bit     |
| -------------- | -------------- | ---------- | -------- |
| 对象的hashCode | 对象的分代年龄 | 是否偏向锁 | 锁标志位 |

<table style="text-align: center">
  <tr>
  	<td rowspan="2">锁状态</td>
    <td colspan="2">25bit</td>
    <td rowspan="2">4bit</td>
    <td>1bit</td>
    <td>2bit</td>
  </tr>
	<tr>
  	<td>23bit</td>
    <td>2bit</td>
    <td>是否是偏向锁</td>
    <td>锁标志位</td>
  </tr>
  <tr>
  	<td>无锁状态</td>
    <td colspan="4">对象hashcode、对象分代年龄</td>
    <td>01</td>
  </tr>
  <tr>
  	<td>轻量级锁</td>
    <td colspan="4">指向锁记录的指针</td>
    <td>00</td>
  </tr>
  <tr>
  	<td>重量级锁</td>
    <td colspan="4">指向重量级锁的指针</td>
    <td>10</td>
  </tr>
  <tr>
  	<td>GC标记</td>
    <td colspan="4">空，不需要记录信息</td>
    <td>11</td>
  </tr>
  <tr>
  	<td>偏向锁</td>
    <td>线程ID</td>
    <td>Epoch</td>
    <td>对象分代年龄</td>
    <td>1</td>
    <td>01</td>
  </tr>
</table>

我们知道我们之前探讨的synchornized关键字使用反编译字节码出现了Monitor,所以我们在看一下monitor的组成

|   Owner   |
| :-------: |
|  EntryQ   |
|  RcThis   |
|   Nest    |
| HashCode  |
| Candidate |

> - **Owner**：初始时为NULL表示当前没有任何线程拥有该monitor record，当线程成功拥有该锁后保存线程唯一标识，当锁被释放时又设置为NULL；
> - **EntryQ**: 关联一个系统互斥锁（semaphore），阻塞所有试图锁住monitor record失败的线程。
> - **RcThis**: 表示blocked或waiting在该monitor record上的所有线程的个数。
> - **Nest**: 用来实现重入锁的计数。
> - **HashCode**: 保存从对象头拷贝过来的HashCode值（可能还包含GC age）。
> - **Candidate**: 用来避免不必要的阻塞或等待线程唤醒，因为每一次只有一个线程能够成功拥有锁，如果每次前一个释放锁的线程唤醒所有正在阻塞或等待的线程，会引起不必要的上下文切换（从阻塞到就绪然后因为竞争锁失败又被阻塞）从而导致性能严重下降。Candidate只有两种可能的值
>   - 0表示没有需要唤醒的线程
>   - 1表示要唤醒一个继任线程来竞争锁。

## 锁优化

### 锁膨胀机制

在经过JDK1.6的锁优化后，synchornized 关键字不再那么笨重了，引入了锁膨胀的机制，机制如下：

无锁状态 -> 偏向锁状态 -> 轻量级锁状态 -> 重量级锁状态

注： 这种机制只可以升级不能降级！

### CAS

轻量级锁的实现基础，全英文名称为： Compare And Swap(CAS 比较并交换)，方法传入(目标数据的内存地址，期望值，如果达到期望值更新值)这三个参数，他的实现结构图如下：

> CAS操作过程：首先读取预期原值A，然后在要更新数据的时候再次读取内存位置V的值，如果该值与预期原值A相匹配，那么处理器会自动将该位置V值更新为新值B；否则一直重试该过程，直到成功。
>
> CAS 操作是抱着乐观的态度进行的，它总是认为自己可以成功完成操作。当多个线程同时使用CAS操作一个变量时，只有一个会胜出，并成功更新，其余均会失败。失败的线程不会被挂起，仅是被告知失败，并且允许再次尝试，直到成功。当然也允许失败的线程放弃操作。

![](http://image.tinx.top/20211129095102.png)

#### 底层实现原理

**CAS的底层实现，是通过`lock cmpxchg`汇编指令来实现的。`cmpxchg`用来实现比较交换操作，`lock`前缀指令用来保证多cpu环境下指令所在缓存行的独占同步，保证了原子性，有序性和可见性。**

原子类的很多方法都是使用CAS操作，以上例中的compareAndSet方法为例子：

```java
public final boolean compareAndSet(int expect, int update) {
    return unsafe.compareAndSwapInt(this, valueOffset, expect, update);
}
```

可以看到内部调用了Unsafe类的compareAndSwapInt方法，Unsafe类中的方法能够通过直接操作字段偏移量来修改字段的值，相当于通过“指针”来操作字段，这个字段的偏移量-“指针”的位置需要自己计算，因此如果计算错误那么会带来一系列错误，比如会导致整个JVM实例崩溃等严重问题。因此Java并不鼓励使用该类，甚至命名为“Unsafe”。Unsafe类是Java实现CAS的基石，但它的作用不仅仅是实现CAS，还有操作直接内存等作用，实际上Unsafe类也是juc包的实现的基石。关于Unsafe的详细理解，感兴趣可以参考：[JUC中的Unsafe类详解与使用案例。](https://blog.csdn.net/weixin_43767015/article/details/104643890)

#### CAS三大问题

##### ABA问题

CAS需要再操作值的时候，检查值有没有发生变化，如果没有发生变化则更新。但是一个值，如果原来为A，变成了B，又变成了A，那么使用CAS进行compare and set的时候，会发现它的值根本没变化过，但实际上是变化过的。

ABA问题的解决思路就是使用版本号，1A->2B->3A，在Atomic包中（JDK5），提供了一个现成的AtomicStampedReference类来解决ABA问题，使用的就是添加版本号的方法。

AtomicStampedReference的compareAndSet()方法定义如下：

```java
public boolean compareAndSet(V   expectedReference,
                             V   newReference,
                             int expectedStamp,
                             int newStamp) {
    Pair<V> current = pair;
    return expectedReference == current.reference && expectedStamp == current.stamp &&
        ((newReference == current.reference && newStamp == current.stamp) ||
         casPair(current, Pair.of(newReference, newStamp)));
}
```

compareAndSet有四个参数，分别表示：预期引用、更新后的引用、预期标志、更新后的标志。源码部门很好理解预期的引用 == 当前引用，预期的标识 == 当前标识，如果更新后的引用和标志和当前的引用和标志相等则直接返回true，否则通过Pair生成一个新的pair对象与当前pair CAS替换。Pair为AtomicStampedReference的内部类，主要用于记录引用和版本戳信息（标识），定义如下：

```java
private static class Pair<T> {
  	// 引用
    final T reference;
  	// 版本戳
    final int stamp;
    private Pair(T reference, int stamp) {
        this.reference = reference;
        this.stamp = stamp;
    }
    static <T> Pair<T> of(T reference, int stamp) {
        return new Pair<T>(reference, stamp);
    }
}

private volatile Pair<V> pair;
```

Pair记录着对象的引用和版本戳，版本戳为int型，保持自增。同时Pair是一个不可变对象，其所有属性全部定义为final，对外提供一个of方法，该方法返回一个新建的Pari对象。pair对象定义为volatile，保证多线程环境下的可见性。在AtomicStampedReference中，大多方法都是通过调用Pair的of方法来产生一个新的Pair对象，然后赋值给变量pair。如set方法：

```java
public void set(V newReference, int newStamp) {
    Pair<V> current = pair;
    if (newReference != current.reference || newStamp != current.stamp)
        this.pair = Pair.of(newReference, newStamp);
}
```

***小例子***

下面我们将通过一个例子可以可以看到AtomicStampedReference和AtomicInteger的区别。我们定义两个线程，线程1负责将100 —> 110 —> 100，线程2执行 100 —>120，看两者之间的区别。

```java
public class Test {
    private static AtomicInteger atomicInteger = new AtomicInteger(100);
    private static AtomicStampedReference atomicStamped = new AtomicStampedReference(100, 1);

    public static void main(String[] args) throws InterruptedException {

        //AtomicInteger
        Thread at1 = new Thread(() -> {
            atomicInteger.compareAndSet(100, 110);
            atomicInteger.compareAndSet(110, 100);
        });

        Thread at2 = new Thread(() -> {
            try {
                TimeUnit.SECONDS.sleep(2);      // at1,执行完
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("AtomicInteger:" + atomicInteger.compareAndSet(100, 120));
        });

        at1.start();
        at2.start();

        at1.join();
        at2.join();

        //AtomicStampedReference

        Thread tsf1 = new Thread(() -> {
            try {
                //让 tsf2先获取stamp，导致预期时间戳不一致
                TimeUnit.SECONDS.sleep(2);
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            // 预期引用：100，更新后的引用：110，预期标识getStamp() 更新后的标识getStamp() + 1
            atomicStamped.compareAndSet(100, 110, atomicStamped.getStamp(), atomicStamped.getStamp() + 1);
            atomicStamped.compareAndSet(110, 100, atomicStamped.getStamp(), atomicStamped.getStamp() + 1);
        });

        Thread tsf2 = new Thread(() -> {
            int stamp = atomicStamped.getStamp();

            try {
                TimeUnit.SECONDS.sleep(2);      //线程tsf1执行完
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println("AtomicStampedReference:" + atomicStamped.compareAndSet(100, 120, stamp, stamp + 1));
        });

        tsf1.start();
        tsf2.start();
    }
}
```

![](http://image.tinx.top/20211129120201.png)

##### 循环时间长开销大

由于线程并不会阻塞，如果CAS自旋长时间不成功，这会给CPU带来非常大的执行开销。

如果JVM能支持处理器提供的pause指令，那么效率会有一定的提升。pause指令有两个作用：第一，它可以延迟流水线执行指令（de-pipeline），使CPU不会消耗过多的执行资源，延迟的时间取决于具体实现的版本，在一些处理器上延迟时间是零；第二，它可以避免在退出循环的时候因内存顺序冲突（Memory Order Violation）而引起CPU流水线被清空（CPU Pipeline Flush），从而提高CPU的执行效率。

##### 只能保证一个共享变量的原子操作

当对一个共享变量执行操作时，我们可以使用循环CAS的方式来保证原子操作，由于CAS底层只能锁定单个指令，均是针对单个变量的操作，对多个共享变量操作时意味着多个指令，此时CAS就无法保证所有操作的原子性，这个时候就可以用锁。

还有一个取巧的办法，就是把多个共享变量合并成一个共享变量来操作。比如，有两个共享变量i＝2，j=a，合并一下ij=2a，然后用CAS来操作ij。从Java 1.5开始，JDK提供了AtomicReference类来保证引用对象之间的原子性，就可以把多个变量放在一个对象里来进行CAS操作。

### 自旋锁

所谓自旋锁，大致的意思就是进入一个死循环状态，当满足某一些条件时，会从这个循环跳出来，目的是解决 <b>线程的阻塞和唤醒需要Cpu从用户态转换为核心态(因为频繁的从用户态转换为内核态是一件负担很重的工作)</b>

在JDK1.6中默认开启，可以通过参数-XX:PreBlockSpin来调整默认自旋次数，但是JDK1.6是自适应自旋锁，所谓自适应就意味着自旋的次数不再是固定的，它是由前一次在同一个锁上的自旋时间及锁的拥有者的状态来决定。

### 锁消除

为了保证数据的完整性，我们在进行操作时需要对这部分操作进行同步设置，但是在有些条件下，JVM检测到不可能存在共享数据竞争，这是JVM会对这部分的同步锁进行消除(不存在竞争，加锁有什么意思呢？)

示例代码如下：

```java
public static void main(String[] args) {
    Map<Integer,Integer> table = new Hashtable<>();
    for (int i = 1; i < 10; i++) {
        table.put(i, i);
    }
    System.out.println(table);
}
```

在运行这段代码时，JVM可以检测到变量table没有方法逃逸的嫌疑(方法逃逸可见JVM分析)，所以JVM可以大胆地将table内部的加synchornized锁操作消除。

### 锁粗化

所谓 锁粗化，就是存在锁激烈竞争的状态下，将多个连续的锁连接在一起，扩大锁范围

### 偏向锁

引入偏向锁主要目的是：为了在无多线程竞争的情况下尽量减少不必要的轻量级锁执行路径。***轻量级锁的加锁解锁操作是需要依赖多次CAS原子指令的***。那么偏向锁是如何来减少不必要的```CAS操作```呢？我们可以查看Mark work的结构就明白了。只需要检查是否为偏向锁、锁标识为以及ThreadID即可，处理流程如下： 

**获取锁**

1. 检测Mark Word是否为可偏向状态，即是否为偏向锁1，锁标识位为01；
2. 若为可偏向状态，则测试线程ID是否为当前线程ID，如果是，则执行步骤（5），否则执行步骤（3）；
3. 如果线程ID不为当前线程ID，则通过CAS操作竞争锁，竞争成功，则将Mark Word的线程ID替换为当前线程ID，否则执行线程（4）；
4. 通过CAS竞争锁失败，证明当前存在多线程竞争情况，当到达全局安全点，获得偏向锁的线程被挂起，偏向锁升级为轻量级锁，然后被阻塞在安全点的线程继续往下执行同步代码块；
5. 执行同步代码块

**释放锁** 

偏向锁的释放采用了一种只有竞争才会释放锁的机制，线程是不会主动去释放偏向锁，需要等待其他线程来竞争。偏向锁的撤销需要等待全局安全点（这个时间点是上没有正在执行的代码）。其步骤如下：

1. 暂停拥有偏向锁的线程，判断锁对象石是否还处于被锁定状态；
2. 撤销偏向锁，恢复到无锁状态（01）或者轻量级锁的状态；
3. 将偏向锁的线程ID设置为NULL，等待下一个偏向锁的竞争

### 轻量级锁

引入轻量级锁的主要目的是在多线程竞争的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。当关闭偏向锁功能或者多个线程竞争偏向锁导致偏向锁升级为轻量级锁，则会尝试获取轻量级锁，其步骤如下： 

**获取锁**

1. 判断当前对象是否处于无锁状态（hashcode、0、01），若是，则JVM首先将在当前线程的栈帧中建立一个名为锁记录（Lock Record）的空间，用于存储锁对象目前的Mark Word的拷贝（官方把这份拷贝加了一个Displaced前缀，即Displaced Mark Word）；否则执行步骤（3）；
2. JVM利用CAS操作尝试将对象的Mark Word更新为指向Lock Record的指正，如果成功表示竞争到锁，则将锁标志位变成00（表示此对象处于轻量级锁状态），执行同步操作；如果失败则执行步骤（3）；
3. 判断当前对象的Mark Word是否指向当前线程的栈帧，如果是则表示当前线程已经持有当前对象的锁，则直接执行同步代码块；否则只能说明该锁对象已经被其他线程抢占了，这时轻量级锁需要膨胀为重量级锁，锁标志位变成10，后面等待的线程将会进入阻塞状态；

**释放锁** 

轻量级锁的释放也是通过CAS操作来进行的，主要步骤如下：

1. 取出在获取轻量级锁保存在Displaced Mark Word中的数据；
2. 用CAS操作将取出的数据替换当前对象的Mark Word中，如果成功，则说明释放锁成功，否则执行（3）；
3. 如果CAS操作替换失败，说明有其他线程尝试获取该锁，则需要在释放锁的同时需要唤醒被挂起的线程。

对于轻量级锁，其性能提升的依据是“对于绝大部分的锁，在整个生命周期内都是不会存在竞争的”，如果打破这个依据则除了互斥的开销外，还有额外的CAS操作，因此在有多线程竞争的情况下，轻量级锁比重量级锁更慢；

### 重量级锁

重量级锁通过对象内部的监视器（monitor）实现，其中monitor的本质是依赖于底层操作系统的Mutex Lock实现，操作系统实现线程之间的切换需要从用户态到内核态的切换，切换成本非常高。

## Synchornized 锁膨胀的过程

> synchronized 同步锁一共具有四种状态：无锁、偏向锁、轻量级锁、重量级锁，他们会随着竞争情况逐渐升级，此过程为不可逆。所以 synchronized 锁膨胀过程其实就是**无锁 → 偏向锁 → 轻量级锁 → 重量级锁**的一个过程。

### 偏向锁

引入偏向锁的主要目的是：为了在无多线程竞争的情况下尽量减少不必须要的轻量级锁执行路径。其实在大多数情况下，锁不仅不存在多线程竞争，而且总是由同一个线程多次获取，所以引入偏向锁就可以减少很多不必要的性能开销和上下文切换。

### 轻量级锁

引入轻量级锁的主要目的是：在多线程竞争不激烈的前提下，减少传统的重量级锁使用操作系统互斥量产生的性能消耗。需要注意的是轻量级锁并不是取代重量级锁，而是在大多数情况下同步块并不会出现严重的竞争情况，所以引入轻量级锁可以减少重量级锁对线程的阻塞带来的开销。

所以偏向锁是认为环境中不存在竞争情况，而轻量级锁则是认为环境中不存在竞争或者竞争不激烈，轻量级锁所以一般都只会有少数几个线程竞争锁对象，其他线程只需要稍微等待（自旋）下就可以获取锁，但是自旋次数有限制，如果超过该次数，则会升级为重量级锁。

下面介绍锁膨胀过程，直接看图(建议从左上角开始观看)：[若图片看不清，请点击此在浏览器查看](http://image.tinx.top/image-20211201100648869.png)

<img src="http://image.tinx.top/image-20211201100648869.png" alt="image-20211201100648869" style="zoom:500%;" />

### 步骤

1. 一个锁对象刚刚开始创建的时候，没有任何线程来访问它，它是可偏向的，它现在认为只可能有一个线程来访问它，所以当第一个线程来访问他的时候，它会偏向这个线程。此时线程状态为无锁状态，锁标志位为 01，此时 Mark Word 如下图：

​		![image-20211201101536570](http://image.tinx.top/image-20211201101536570.png)

2. 当一个线程（线程 A）来获取锁的时，会首先检查所标志位，此时锁标志位为 01，然后检查是否为偏向锁，此时不为偏向锁，所以当前线程会修改对象头状态为偏向锁，同时将对象头中的 ThreadID 改成自己的 Thread ID，此时 Mark Word 如下图

   ![image-20211201101602072](http://image.tinx.top/image-20211201101602072.png)

3. 如果再有一个线程（线程 B）过来，此时锁状态为偏向锁，该线程会检查 Mark Word 中记录的线程 ID 是否为自己的线程 ID，如果是，则获取偏向锁，执行同步代码块。如果不是，则利用 CAS 尝试替换 Mark Word 中的 Thread ID，成功，表示该线程（线程 B）获取偏向锁，执行同步代码块，此时 Mark Word 如下图：

   ![image-20211201101631117](http://image.tinx.top/image-20211201101631117.png)

4. 如果失败，则表明当前环境存在锁竞争情况，则执行偏向锁的撤销工作（**这里有一点需要注意的是：偏向锁的释放并不是主动，而是被动的，什么意思呢？就是说持有偏向锁的线程执行完同步代码后不会主动释放偏向锁，而是等待其他线程来竞争才会释放锁**）。撤销偏向锁的操作需要等到全局安全点才会执行，然后暂停持有偏向锁的线程，同时检查该线程的状态，如果该线程不处于活动状态或者已经退出同步代码块，则设置为无锁状态（线程 ID 为空，是否为偏向锁为 0 ，锁标志位为01）重新偏向，同时恢复该线程。若该线程活着，则会遍历该线程栈帧中的锁记录，检查锁记录的使用情况，如果仍然需要持有偏向锁，则撤销偏向锁，升级为轻量级锁。

5. 在升级为轻量级锁之前，持有偏向锁的线程（线程 A）是暂停的，JVM 首先会在原持有偏向锁的线程（线程 A）的栈中创建一个名为锁记录的空间（Lock Record），用于存放锁对象目前的 Mark Word 的拷贝，然后拷贝对象头中的 Mark Word 到原持有偏向锁的线程（线程 A）的锁记录中（官方称之为 Displaced Mark Word ），这时线程 A 获取轻量级锁，此时 Mark Word 的锁标志位为 00，指向锁记录的指针指向线程 A 的锁记录地址，如下图：

   <img src="http://image.tinx.top/image-20211201101707318.png" alt="image-20211201101707318" style="zoom: 50%;" />

6. 当原持有偏向锁的线程（线程 A）获取轻量级锁后，JVM 唤醒线程 A，线程 A 执行同步代码块，执行完成后，开始轻量级锁的释放过程。

7. 对于其他线程而言，也会在栈帧中建立锁记录，存储锁对象目前的 Mark Word 的拷贝。JVM 利用 CAS 操作尝试将锁对象的 Mark Word 更正指向当前线程的 Lock Record，如果成功，表明竞争到锁，则执行同步代码块，如果失败，那么线程尝试使用自旋的方式来等待持有轻量级锁的线程释放锁。当然，它不会一直自旋下去，因为自旋的过程也会消耗 CPU，而是自旋一定的次数，如果自旋了一定次数后还是失败，则升级为重量级锁，阻塞所有未获取锁的线程，等待释放锁后唤醒。

8. 轻量级锁的释放，会使用 CAS 操作将 Displaced Mark Word 替换会对象头中，成功，则表示没有发生竞争，直接释放。如果失败，表明锁对象存在竞争关系，这时会轻量级锁会升级为重量级锁，然后释放锁，唤醒被挂起的线程，开始新一轮锁竞争，注意这个时候的锁是重量级锁。



# Happends-before

> 在JMM中，如果一个操作执行的结果需要对另一个操作可见，那么这两个操作之间必须存在happens-before关系。

如果有以下代码：

```java
i = 1; // 线程1
j = 1; // 线程2
```

j 是否等于1呢？假定线程A的操作（i = 1）happens-before线程B的操作（j = i）,那么可以确定线程B执行后j = 1 一定成立，如果他们不存在happens-before原则，那么j = 1 不一定成立。这就是happens-before原则的威力。 happens-before原则定义如下： **1. 如果一个操作happens-before另一个操作，那么第一个操作的执行结果将对第二个操作可见，而且第一个操作的执行顺序排在第二个操作之前。** **2. 两个操作之间存在happens-before关系，并不意味着一定要按照happens-before原则制定的顺序来执行。如果重排序之后的执行结果与按照happens-before关系来执行的结果一致，那么这种重排序并不非法。** 

> 下面是happens-before原则规则：
>
> 1. 程序次序规则：一个线程内，按照代码顺序，书写在前面的操作先行发生于书写在后面的操作；
> 2. 锁定规则：一个unLock操作先行发生于后面对同一个锁的lock操作；
> 3. volatile变量规则：对一个变量的写操作先行发生于后面对这个变量的读操作；
> 4. 传递规则：如果操作A先行发生于操作B，而操作B又先行发生于操作C，则可以得出操作A先行发生于操作C；
> 5. 线程启动规则：Thread对象的start()方法先行发生于此线程的每个一个动作；
> 6. 线程中断规则：对线程interrupt()方法的调用先行发生于被中断线程的代码检测到中断事件的发生；
> 7. 线程终结规则：线程中所有的操作都先行发生于线程的终止检测，我们可以通过Thread.join()方法结束、Thread.isAlive()的返回值手段检测到线程已经终止执行；
> 8. 对象终结规则：一个对象的初始化完成先行发生于他的finalize()方法的开始；
>
> 我们来详细看看上面每条规则（摘自《深入理解Java虚拟机第12章》）：
>
> - **程序次序规则**：一段代码在单线程中执行的结果是有序的。注意是执行结果，因为虚拟机、处理器会对指令进行重排序（重排序后面会详细介绍）。虽然重排序了，但是并不会影响程序的执行结果，所以程序最终执行的结果与顺序执行的结果是一致的。故而这个规则只对单线程有效，在多线程环境下无法保证正确性。
> - **volatile变量规则**：这是一条比较重要的规则，它标志着volatile保证了线程可见性。通俗点讲就是如果一个线程先去写一个volatile变量，然后一个线程去读这个变量，那么这个写操作一定是happens-before读操作的。
> - **传递规则**：提现了happens-before原则具有传递性，即A happens-before B , B happens-before C，那么A happens-before C
> - **线程启动规则**：假定线程A在执行过程中，通过执行ThreadB.start()来启动线程B，那么线程A对共享变量的修改在接下来线程B开始执行后确保对线程B可见。
> - **线程终结规则**：假定线程A在执行的过程中，通过制定ThreadB.join()等待线程B终止，那么线程B在终止之前对共享变量的修改在线程A等待返回后可见。
>
> 上面八条是原生Java满足Happens-before关系的规则，但是我们可以对他们进行推导出其他满足happens-before的规则：
>
> 1. 将一个元素放入一个线程安全的队列的操作Happens-Before从队列中取出这个元素的操作
> 2. 将一个元素放入一个线程安全容器的操作Happens-Before从容器中取出这个元素的操作
> 3. 在CountDownLatch上的倒数操作Happens-Before CountDownLatch#await()操作
> 4. 释放Semaphore许可的操作Happens-Before获得许可操作
> 5. Future表示的任务的所有操作Happens-Before Future#get()操作
> 6. 向Executor提交一个Runnable或Callable的操作Happens-Before任务开始执行操作

## 重排序

> 在执行程序时，为了提供性能，处理器和编译器常常会对指令进行重排序，但是不能随意重排序，不是你想怎么排序就怎么排序，它需要满足以下两个条件：
>
> 1. 在单线程环境下不能改变程序运行的结果；
> 2. 存在数据依赖关系的不允许重排序

### as-if-serial语义

as-if-serial语义的意思是，所有的操作均可以为了优化而被重排序，但是你必须要保证重排序后执行的结果不能被改变，编译器、runtime、处理器都必须遵守as-if-serial语义。注意as-if-serial只保证单线程环境，多线程环境下无效。

### happends-before与as-if-serial区别

+ As-if-serial 语义仅限于单线程，保证单线程不被重排序
+ Happends-before 语义可以在单线程也可以在多线程，保证代码执行不被重排序

# Volatile

> 读音： [ˈvɒlətaɪl] 
>
> **volatile可以保证线程可见性且提供了一定的有序性，但是无法保证原子性。**在JVM底层volatile是采用“内存屏障”来实现的。
>
> + ***内存屏障可以禁止指令重排序***
> + ***保证可见性、不保证原子性***
>
> 如果我们要执行 ```i++```的操作，当线程运行这段代码时，
>
> 1. 首先会从主存中读取i( i = 1)，然后复制一份到CPU高速缓存中，
> 2. 然后CPU执行 + 1 的操作，
> 3. 然后将数据（2）写入到告诉缓存中，最后刷新到主存中。
>
> 其实这样做在单线程中是没有问题的，有问题的是在多线程中。如下： 假如有两个线程A、B都执行这个操作（i++），按照我们正常的逻辑思维主存中的i值应该=3，但事实是这样么？分析如下： 两个线程从主存中读取i的值（1）到各自的高速缓存中，然后线程A执行+1操作并将结果写入高速缓存中，最后写入主存中，此时主存i==2,线程B做同样的操作，主存中的i仍然=2。所以最终结果为2并不是3。这种现象就是缓存一致性问题。 解决缓存一致性方案有两种：
>
> 1. 通过在总线加LOCK#锁的方式
>
>    存在一个问题，它是采用一种独占的方式来实现的，即总线加LOCK#锁的话，只能有一个CPU能够运行，其他CPU都得阻塞，效率较为低下。 
>
> 2. 通过缓存一致性协议
>
>    缓存一致性协议（MESI协议）它确保每个缓存中使用的共享变量的副本是一致的。其核心思想如下：当某个CPU在写数据时，如果发现操作的变量是共享变量，则会通知其他CPU告知该变量的缓存行是无效的，因此其他CPU在读取该变量时，发现其无效会重新从主存中加载数据。

在并发编程中，我们都会遇到如下的基本概念：

1. 原子性

   何为原子性？一个操作或者多个操作 要么全部执行并且执行的过程不会被任何因素打断，要么就都不执行。

2. 可见性

   可见性是指当多个线程访问同一个变量时，一个线程修改了这个变量的值，其他线程能够立即看得到修改的值。

3. 有序性

   程序执行的顺序按照代码的先后顺序执行。



## 使用场景

+ 状态标记变量

+ double check(单例设计模式的双重检查)

  ```java
  public class Singleton{
    private static volatile Singleton single;
    
    private Singleton(){}
    
    public static Singleton getInstance(){
      if(single == null){
        synchornized(Singleton.class){
          if(single == null){
            single = new Singleton();
          }
        }
      }
      return single;
    } 
  }
  ```
  
  

# AQS

> AQS，AbstractQueuedSynchronizer，即队列同步器，它是构建锁或者其他同步组件的基础框架（如ReentrantLock、ReentrantReadWriteLock、Semaphore等）

AQS主要提供了如下一些方法：

- `getState()`：返回同步状态的当前值；
- `setState(int newState)`：设置当前同步状态；
- `compareAndSetState(int expect, int update)`：使用CAS设置当前状态，该方法能够保证状态设置的原子性；
- `tryAcquire(int arg)`：独占式获取同步状态，获取同步状态成功后，其他线程需要等待该线程释放同步状态才能获取同步状态；
- `tryRelease(int arg)`：独占式释放同步状态；
- `tryAcquireShared(int arg)`：共享式获取同步状态，返回值大于等于0则表示获取成功，否则获取失败；
- `tryReleaseShared(int arg)`：共享式释放同步状态；
- `isHeldExclusively()`：当前同步器是否在独占式模式下被线程占用，一般该方法表示是否被当前线程所独占；
- `acquire(int arg)`：独占式获取同步状态，如果当前线程获取同步状态成功，则由该方法返回，否则，将会进入同步队列等待，该方法将会调用可重写的tryAcquire(int arg)方法；
- `acquireInterruptibly(int arg)`：与acquire(int arg)相同，但是该方法响应中断，当前线程为获取到同步状态而进入到同步队列中，如果当前线程被中断，则该方法会抛出InterruptedException异常并返回；
- `tryAcquireNanos(int arg,long nanos)`：超时获取同步状态，如果当前线程在nanos时间内没有获取到同步状态，那么将会返回false，已经获取则返回true；
- `acquireShared(int arg)`：共享式获取同步状态，如果当前线程未获取到同步状态，将会进入同步队列等待，与独占式的主要区别是在同一时刻可以有多个线程获取到同步状态；
- `acquireSharedInterruptibly(int arg)`：共享式获取同步状态，响应中断；
- `tryAcquireSharedNanos(int arg, long nanosTimeout)`：共享式获取同步状态，增加超时限制；
- `release(int arg)`：独占式释放同步状态，该方法会在释放同步状态之后，将同步队列中第一个节点包含的线程唤醒；
- `releaseShared(int arg)`：共享式释放同步状态；

## CLH同步队列

> AQS内部维护着一个FIFO队列，该队列就是CLH同步队列。 
>
> CLH同步队列是一个FIFO双向队列，AQS依赖它来完成同步状态的管理，当前线程如果获取同步状态失败时，AQS则会将当前线程已经等待状态等信息构造成一个节点（Node）并将其加入到CLH同步队列，同时会阻塞当前线程，当同步状态释放时，会把首节点唤醒（公平锁），使其再次尝试获取同步状态。 
>
> 在CLH同步队列中，一个节点表示一个线程，它保存着线程的引用（thread）、状态（waitStatus）、前驱节点（prev）、后继节点（next），其定义如下：

```java
static final class Node {
   /** 共享 */
   static final Node SHARED = new Node();
    
   /** 独占 */
   static final Node EXCLUSIVE = null;
    
   /**
    * 因为超时或者中断，节点会被设置为取消状态，被取消的节点时不会参与到竞争中的，他会一直保持取消状态不会转变为其他状态；
    */
   static final int CANCELLED =  1;
    
   /**
    * 后继节点的线程处于等待状态，而当前节点的线程如果释放了同步状态或者被取消，将会通知后继节点，使后继节点的线程得以运行
    */
   static final int SIGNAL    = -1;
    
   /**
    * 节点在等待队列中，节点线程等待在Condition上，当其他线程对Condition调用了signal()后，改节点将会从等待队列中转移到同步队列中，加入到同步状态的获取中
    */
   static final int CONDITION = -2;
    
   /**
    * 表示下一次共享式同步状态获取将会无条件地传播下去
    */
   static final int PROPAGATE = -3;
    
   /** 等待状态 */
   volatile int waitStatus;
    
   /** 前驱节点 */
   volatile Node prev;
    
   /** 后继节点 */
   volatile Node next;
    
   /** 获取同步状态的线程 */
   volatile Thread thread;
    
   Node nextWaiter;
    
   final boolean isShared() {
       return nextWaiter == SHARED;
   }
    
   final Node predecessor() throws NullPointerException {
       Node p = prev;
       if (p == null) throw new NullPointerException();
       else return p;
   }
    
   Node(Thread thread, Node mode) {
       this.nextWaiter = mode;
       this.thread = thread;
   }
    
   Node(Thread thread, int waitStatus) {
       this.waitStatus = waitStatus;
       this.thread = thread;
   }
}
```

![](http://image.tinx.top/20211126173146.png)





## 同步状态的获取与释放

### 独占式

#### 独占式同步状态获取

acquire(int arg)方法为AQS提供的模板方法，该方法为独占式获取同步状态，但是该方法对中断不敏感，也就是说由于线程获取同步状态失败加入到CLH同步队列中，后续对线程进行中断操作时，线程不会从同步队列中移除。代码如下:

```java
public final void acquire(int arg) {
    if (!tryAcquire(arg) &&
        acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
        selfInterrupt();
}
```

各个方法定义如下：

1. tryAcquire：去尝试获取锁，获取成功则设置锁状态并返回true，否则返回false。该方法自定义同步组件自己实现，该方法必须要保证线程安全的获取同步状态。
2. addWaiter：如果tryAcquire返回FALSE（获取同步状态失败），则调用该方法将当前线程加入到CLH同步队列尾部。
3. acquireQueued：当前线程会根据公平性原则来进行阻塞等待（自旋）,直到获取锁为止；并且返回当前线程在等待过程中有没有中断过。
4. selfInterrupt：产生一个中断

acquireQueued方法为一个自旋的过程，也就是说当前线程（Node）进入同步队列后，就会进入一个自旋的过程，每个节点都会自省地观察，当条件满足，获取到同步状态后，就可以从这个自旋过程中退出，否则会一直执行下去。如下：

```java
final boolean acquireQueued(final Node node, int arg) {
    boolean failed = true;
    try {
        //中断标志
        boolean interrupted = false;
        /*
         * 自旋过程，其实就是一个死循环而已
         */
        for (;;) {
            //当前线程的前驱节点
            final Node p = node.predecessor();
            //当前线程的前驱节点是头结点，且同步状态成功
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return interrupted;
            }
            //获取失败，线程等待--具体后面介绍
            if (shouldParkAfterFailedAcquire(p, node) &&
                    parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

从上面代码中可以看到，当前线程会一直尝试获取同步状态，当然前提是只有其前驱节点为头结点才能够尝试获取同步状态，理由：

1. 保持FIFO同步队列原则。
2. 头节点释放同步状态后，将会唤醒其后继节点，后继节点被唤醒后需要检查自己是否为头节点。

acquire(int arg)方法流程图如下：

![](http://image.tinx.top/20211126174112.png)

#### 独占式获取响应中断

AQS提供了acquire(int arg)方法以供独占式获取同步状态，但是该方法对中断不响应，对线程进行中断操作后，该线程会依然位于CLH同步队列中等待着获取同步状态。为了响应中断，AQS提供了acquireInterruptibly(int arg)方法，该方法在等待获取同步状态时，如果当前线程被中断了，会立刻响应中断抛出异常InterruptedException

```java
public final void acquireInterruptibly(int arg)
        throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    if (!tryAcquire(arg)) doAcquireInterruptibly(arg);
}
```

首先校验该线程是否已经中断了，如果是则抛出InterruptedException，否则执行tryAcquire(int arg)方法获取同步状态，如果获取成功，则直接返回，否则执行doAcquireInterruptibly(int arg)。doAcquireInterruptibly(int arg)定义如下：

```java
private void doAcquireInterruptibly(int arg)
    throws InterruptedException {
    final Node node = addWaiter(Node.EXCLUSIVE);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head && tryAcquire(arg)) {
                setHead(node);
                p.next = null; // help GC
                failed = false;
                return;
            }
            if (shouldParkAfterFailedAcquire(p, node) &&
                parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

doAcquireInterruptibly(int arg)方法与acquire(int arg)方法仅有两个差别。1.方法声明抛出InterruptedException异常，2.在中断方法处不再是使用interrupted标志，而是直接抛出InterruptedException异常

```java
// acquireInterruptibly -> doAcquireInterruptibly
if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
    throw new InterruptedException();
// tryAccquire -> acquireQueued 获取失败，线程等待--具体后面介绍
if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
    interrupted = true;
```

#### 独占式同步状态释放

当线程获取同步状态后，执行完相应逻辑后就需要释放同步状态。AQS提供了release(int arg)方法释放同步状态：

```java
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

该方法同样是先调用自定义同步器自定义的tryRelease(int arg)方法来释放同步状态，释放成功后，会调用unparkSuccessor(Node node)方法唤醒后继节点

> 在AQS中维护着一个FIFO的同步队列，当线程获取同步状态失败后，则会加入到这个CLH同步队列的队尾并一直保持着自旋。在CLH同步队列中的线程在自旋时会判断其前驱节点是否为首节点，如果为首节点则不断尝试获取同步状态，获取成功则退出CLH同步队列。当线程执行完逻辑后，会释放同步状态，释放后会唤醒其后继节点。

### 共享式

共享式与独占式的最主要区别在于同一时刻独占式只能有一个线程获取同步状态，而共享式在同一时刻可以有多个线程获取同步状态。例如读操作可以有多个线程同时进行，而写操作同一时刻只能有一个线程进行写操作，其他操作都会被阻塞。

#### 共享式同步状态获取

AQS提供acquireShared(int arg)方法共享式获取同步状态：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        //获取失败，自旋获取同步状态
        doAcquireShared(arg);
}
```

从上面程序可以看出，方法首先是调用tryAcquireShared(int arg)方法尝试获取同步状态，如果获取失败则调用doAcquireShared(int arg)自旋方式获取同步状态，共享式获取同步状态的标志是返回 >= 0 的值表示获取成功。自选式获取同步状态如下：

```java
private void doAcquireShared(int arg) {
    // 共享式节点
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        boolean interrupted = false;
        for (;;) {
            //前驱节点
            final Node p = node.predecessor();
            //如果其前驱节点，获取同步状态
            if (p == head) {
                //尝试获取同步
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    if (interrupted)
                        selfInterrupt();
                    failed = false;
                    return;
                }
            }
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                interrupted = true;
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

tryAcquireShared(int arg)方法尝试获取同步状态，返回值为int，当其 >= 0 时，表示能够获取到同步状态，这个时候就可以从自旋过程中退出。 

acquireShared(int arg)方法不响应中断，与独占式相似，AQS也提供了响应中断、超时的方法，分别是：acquireSharedInterruptibly(int arg)、tryAcquireSharedNanos(int arg,long nanos)。

#### 共享式同步状态释放

获取同步状态后，需要调用release(int arg)方法释放同步状态，方法如下：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

因为可能会存在多个线程同时进行释放同步状态资源，所以需要确保同步状态安全地成功释放，一般都是通过CAS和循环来完成的。



# ReentrantLock

> ReentrantLock，可重入锁，是一种递归无阻塞的同步机制。它可以等同于synchronized的使用，但是ReentrantLock提供了比synchronized更强大、灵活的锁机制，可以减少死锁发生的概率。 API介绍如下：
>
> 一个可重入的互斥锁定 Lock，它具有与使用 synchronized 方法和语句所访问的隐式监视器锁定相同的一些基本行为和语义，但功能更强大。ReentrantLock 将由最近成功获得锁定，并且还没有释放该锁定的线程所拥有。当锁定没有被另一个线程所拥有时，调用 lock 的线程将成功获取该锁定并返回。如果当前线程已经拥有该锁定，此方法将立即返回。可以使用 isHeldByCurrentThread() 和 getHoldCount() 方法来检查此情况是否发生。

ReentrantLock还提供了公平锁也非公平锁的选择，构造方法接受一个可选的公平参数（默认非公平锁），当设置为true时，表示公平锁，否则为非公平锁。公平锁与非公平锁的区别在于公平锁的锁获取是有顺序的。但是公平锁的效率往往没有非公平锁的效率高，在许多线程访问的情况下，公平锁表现出较低的吞吐量。

![](https://www.cmsblogs.com/images/group/sike-java/sike-java-bingfa/202105091543502041.png)

## 获取锁

```java
ReentrantLock lock = new ReentrantLock();
lock.lock();
```

那么lock底层是如何实现的呢？让我们追寻源码去一探究竟：

![](http://image.tinx.top/20211128224324.png)

进入源码，我们可以看到直接调用了ReentrantLock内部的sync对象(抽象的Sync内部类继承了 ***AbstractQueuedSynchronizer*** AQS对象)的lock方法，我们看到这是一个抽象类的方法，然后下面的具体实现就是 公平锁和非公平锁来实现了具体的代码，这就是我们在一开始介绍中的 ```ReenrantLock``` 可以使用公平锁或者非公平锁的关键！

![](http://image.tinx.top/20211128224606.png)

我们先来看一下无参构造函数默认实现非公平的锁的源码：

```java
final void lock() {
    //尝试使用CAS方式设置状态值，目的是获取锁
    if (compareAndSetState(0, 1))
      	// 获取成功，就是将这个线程设置为获取锁的线程
        setExclusiveOwnerThread(Thread.currentThread());
    else
        //获取失败，调用AQS的acquire(int arg)方法
        acquire(1);
}
```

首先会第一次尝试快速获取锁，如果获取失败，则调用acquire(int arg)方法，该方法定义在AQS中，如下：

```java
public final void acquire(int arg) {
  	// 首先先尝试自旋的方式获取锁，如果失败
  	// 就将本线程放入一个等待队列中
    if (!tryAcquire(arg) && acquireQueued(addWaiter(Node.EXCLUSIVE), arg))
      	// 然后暂时性的中断这个线程
        selfInterrupt();
}

static void selfInterrupt() {
    Thread.currentThread().interrupt();
}
```

这个方法首先调用tryAcquire(int arg)方法，在AQS中讲述过，tryAcquire(int arg)需要自定义同步组件提供实现，非公平锁实现如下：

```java
protected final boolean tryAcquire(int acquires) {
  	// 使用非公平的锁
    return nonfairTryAcquire(acquires);
}

final boolean nonfairTryAcquire(int acquires) {
    //当前线程
    final Thread current = Thread.currentThread();
    //获取同步状态
    int c = getState();
    //state == 0,表示没有该锁处于空闲状态
    if (c == 0) {
        //获取锁成功，设置为当前线程所有
        if (compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    //线程重入
    //判断锁持有的线程是否为当前线程
    else if (current == getExclusiveOwnerThread()) {
      	// 增加 acquires 数量的锁个数，供后面使用
        int nextc = c + acquires;
      	// 如果计算出来的结果小于0 肯定就是不正常了，会进行错误抛出
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
      	// 设置锁的个数
        setState(nextc);
        return true;
    }
    return false;
}
```

该方法主要逻辑：

1. 首先判断同步状态state == 0 ? 如果是表示该锁还没有被线程持有，直接通过CAS获取同步状态，如果成功返回true。
2. 如果state != 0，则判断当前线程是否为获取锁的线程，如果是则获取锁，成功返回true。成功获取锁的线程再次获取锁，这是增加了同步状态state。

## 释放锁

获取同步锁后，使用完毕则需要释放锁，ReentrantLock提供了unlock释放锁：

```java
public void unlock() {
      sync.release(1);
}
```

unlock内部使用Sync的release(int arg)释放锁，release(int arg)是在AQS中定义的：

```java
public final boolean release(int arg) {
  	// 尝试去释放锁
    if (tryRelease(arg)) {
      	// 拿到头结点
        Node h = head;
      	// 如果头结点不为空 并且 头结点的 等待状态标志不为0
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

与获取同步状态的acquire(int arg)方法相似，释放同步状态的tryRelease(int arg)同样是需要自定义同步组件自己实现：

```java
protected final boolean tryRelease(int releases) {
    //减掉releases
    int c = getState() - releases;
    //如果释放的不是持有锁的线程，抛出异常
    if (Thread.currentThread() != getExclusiveOwnerThread()) throw new IllegalMonitorStateException();
    boolean free = false;
    //state == 0 表示已经释放完全了，其他线程可以获取同步状态了
    if (c == 0) {
        free = true;
        setExclusiveOwnerThread(null);
    }
    setState(c);
    return free;
}
```

只有当同步状态彻底释放后该方法才会返回true。当state == 0 时，则将锁持有线程设置为null，free= true，表示释放成功。

## 公平锁和非公平锁的区别

> 公平锁与非公平锁的区别在于获取锁的时候是否按照FIFO的顺序来。释放锁不存在公平性和非公平性，上面以非公平锁为例，下面我们来看看公平锁的tryAcquire(int arg)：

```java
protected final boolean tryAcquire(int acquires) {
    final Thread current = Thread.currentThread();
    int c = getState();
    if (c == 0) {
        if (!hasQueuedPredecessors() && compareAndSetState(0, acquires)) {
            setExclusiveOwnerThread(current);
            return true;
        }
    }
    else if (current == getExclusiveOwnerThread()) {
        int nextc = c + acquires;
        if (nextc < 0) throw new Error("Maximum lock count exceeded");
        setState(nextc);
        return true;
    }
    return false;
}
```

比较非公平锁和公平锁获取同步状态的过程，会发现两者唯一的区别就在于公平锁在获取同步状态时多了一个限制条件：hasQueuedPredecessors()，定义如下：

```java
public final boolean hasQueuedPredecessors() {
    Node t = tail;  //尾节点
    Node h = head;  //头节点
    Node s;
    //头节点 != 尾节点
    //同步队列第一个节点不为null
    //当前线程是同步队列第一个节点
    return h != t && ((s = h.next) == null || s.thread != Thread.currentThread());
}
```

该方法主要做一件事情：***主要是判断当前线程是否位于CLH同步队列中的第一个。如果是则返回true，否则返回false。***

## ReentrantLock 和 Synchornized 区别

前面提到ReentrantLock提供了比synchronized更加灵活和强大的锁机制，那么它的灵活和强大之处在哪里呢？他们之间又有什么相异之处呢？ 首先他们肯定具有相同的功能和内存语义。

1. 与synchronized相比，ReentrantLock提供了更多，更加全面的功能，具备更强的扩展性。例如：时间锁等候，可中断锁等候，锁投票。
2. ReentrantLock还提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活，所以在多个条件变量和高度竞争锁的地方，ReentrantLock更加适合（以后会阐述Condition）。
3. ReentrantLock提供了可轮询的锁请求。它会尝试着去获取锁，如果成功则继续，否则可以等到下次运行时处理，而synchronized则一旦进入锁请求要么成功要么阻塞，所以相比synchronized而言，ReentrantLock会不容易产生死锁些。
4. ReentrantLock支持更加灵活的同步代码块，但是使用synchronized时，只能在同一个synchronized块结构中获取和释放。注：ReentrantLock的锁释放一定要在finally中处理，否则可能会产生严重的后果。
5. ReentrantLock支持中断处理，且性能较synchronized会好些。

# ReentrantReadWriteLock

重入锁ReentrantLock是排他锁，排他锁在同一时刻仅有一个线程可以进行访问，但是在大多数场景下，大部分时间都是提供读服务，而写服务占有的时间较少。然而读服务不存在数据竞争问题，如果一个线程在读时禁止其他线程读势必会导致性能降低。所以就提供了读写锁。 读写锁维护着一对锁，一个读锁和一个写锁。通过分离读锁和写锁，使得并发性比一般的排他锁有了较大的提升：在同一时间可以允许多个读线程同时访问，但是在写线程访问时，所有读线程和写线程都会被阻塞。 读写锁的主要特性：

1. 公平性：支持公平性和非公平性。
2. 重入性：支持重入。读写锁最多支持65535个递归写入锁和65535个递归读取锁。
3. 锁降级：遵循获取写锁、获取读锁在释放写锁的次序，写锁能够降级成为读锁

读写锁ReentrantReadWriteLock实现接口ReadWriteLock，该接口维护了一对相关的锁，一个用于只读操作，另一个用于写入操作。只要没有 writer，读取锁可以由多个 reader 线程同时保持。写入锁是独占的。

```java
// 读写锁内部有两把锁
public interface ReadWriteLock {
  	// 第一把锁就是 读锁 共享的
    Lock readLock();
  	// 第二把锁就是 写锁 互斥的
    Lock writeLock();
}
```

ReadWriteLock定义了两个方法。readLock()返回用于读操作的锁，writeLock()返回用于写操作的锁。ReentrantReadWriteLock定义如下：

```java
/** 内部类  读锁 */
private final ReentrantReadWriteLock.ReadLock readerLock;
/** 内部类  写锁 */
private final ReentrantReadWriteLock.WriteLock writerLock;

final Sync sync;

/** 使用默认（非公平）的排序属性创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock() {
    this(false);
}

/** 使用给定的公平策略创建一个新的 ReentrantReadWriteLock */
public ReentrantReadWriteLock(boolean fair) {
    sync = fair ? new FairSync() : new NonfairSync();
    readerLock = new ReadLock(this);
    writerLock = new WriteLock(this);
}

/** 返回用于写入操作的锁 */
public ReentrantReadWriteLock.WriteLock writeLock() { return writerLock; }
/** 返回用于读取操作的锁 */
public ReentrantReadWriteLock.ReadLock  readLock()  { return readerLock; }

abstract static class Sync extends AbstractQueuedSynchronizer {
    /**
     * 省略其余源代码
     */
}
public static class WriteLock implements Lock, java.io.Serializable{
    /**
     * 省略其余源代码
     */
}

public static class ReadLock implements Lock, java.io.Serializable {
    /**
     * 省略其余源代码
     */
}
```

ReentrantReadWriteLock与ReentrantLock一样，其锁主体依然是Sync，它的读锁、写锁都是依靠Sync来实现的。所以ReentrantReadWriteLock实际上只有一个锁，只是在获取读取锁和写入锁的方式上不一样而已，它的读写锁其实就是两个类：ReadLock、WriteLock，这两个类都是lock实现。 **在ReentrantLock中使用一个int类型的state来表示同步状态**，该值表示锁被一个线程重复获取的次数。

但是读写锁ReentrantReadWriteLock内部维护着两个一对锁，需要用一个变量维护多种状态。所以读写锁采用“按位切割使用”的方式来维护这个变量，将其切分为两部分，高16为表示读，低16为表示写。

分割之后，读写锁是如何迅速确定读锁和写锁的状态呢？通过为运算。假如当前同步状态为S，那么写状态等于 S & 0x0000FFFF（将高16位全部抹去），读状态等于S >>> 16(无符号补0右移16位)。代码如下：

```java
// 切割的标志 将一个32位整数按 16：16的方式切割，高16位为读 低16位为写
static final int SHARED_SHIFT   = 16;
// 共享单元 65536
static final int SHARED_UNIT    = (1 << SHARED_SHIFT);
// 最大支持所得数量计数为 65535
static final int MAX_COUNT      = (1 << SHARED_SHIFT) - 1;
// 独立掩码 65535，用作求模运算
static final int EXCLUSIVE_MASK = (1 << SHARED_SHIFT) - 1;
// 共享数量
static int sharedCount(int c)    { return c >>> SHARED_SHIFT; }
// 执行的数量 &可以想象是 进行取模%运算，但是计算机进行位运算更快，所以选择 与位运算
static int exclusiveCount(int c) { return c & EXCLUSIVE_MASK; }
```

## 写锁

写锁就是一个支持可重入的排他锁。

### 写锁的获取

写锁的获取最终会调用tryAcquire(int arg)，该方法在内部类Sync中实现：

```java
protected final boolean tryAcquire(int acquires) {
    Thread current = Thread.currentThread();
    // 当前锁个数
    int c = getState();
    // 写锁
    int w = exclusiveCount(c);
  	// 如果锁的个数不为0
    if (c != 0) {
        //c != 0 && w == 0 表示存在读锁
        //当前线程不是已经获取写锁的线程
        if (w == 0 || current != getExclusiveOwnerThread())
            return false;
        //超出最大范围
        if (w + exclusiveCount(acquires) > MAX_COUNT) throw new Error("Maximum lock count exceeded");
      
        setState(c + acquires);
        return true;
    }
    //是否需要阻塞
    if (writerShouldBlock() || !compareAndSetState(c, c + acquires))
        return false;
    //设置获取锁的线程为当前线程
    setExclusiveOwnerThread(current);
    return true;
}
```

该方法和ReentrantLock的tryAcquire(int arg)大致一样，在判断重入时增加了一项条件：读锁是否存在。因为要确保写锁的操作对读锁是可见的，如果在存在读锁的情况下允许获取写锁，那么那些已经获取读锁的其他线程可能就无法感知当前写线程的操作。因此只有等读锁完全释放后，写锁才能够被当前线程所获取，一旦写锁获取了，所有其他读、写线程均会被阻塞。

### 写锁的释放

获取了写锁用完了则需要释放，WriteLock提供了unlock()方法释放写锁：

```java
// 解锁 的源码
public void unlock() {
    sync.release(1);
}

// sync实现的 release
public final boolean release(int arg) {
    if (tryRelease(arg)) {
        Node h = head;
        if (h != null && h.waitStatus != 0)
            unparkSuccessor(h);
        return true;
    }
    return false;
}
```

写锁的释放最终还是会调用AQS的模板方法release(int arg)方法，该方法首先调用tryRelease(int arg)方法尝试释放锁，tryRelease(int arg)方法为读写锁内部类Sync中定义了，如下：

```java
protected final boolean tryRelease(int releases) {
    //释放的线程不为锁的持有者
    if (!isHeldExclusively()) throw new IllegalMonitorStateException();
  	// 计算出新的释放后的值
    int nextc = getState() - releases;
    // 若写锁的新线程数为0，则将锁的持有者设置为null
    boolean free = exclusiveCount(nextc) == 0;
  	// 如果空闲了，就将正在运行的线程置为 null
    if (free) setExclusiveOwnerThread(null);
  	// 设置新的计算值
    setState(nextc);
    return free;
}
```

写锁释放锁的整个过程和独占锁ReentrantLock相似，每次释放均是减少写状态，当写状态为0时表示 写锁已经完全释放了，从而等待的其他线程可以继续访问读写锁，获取同步状态，同时此次写线程的修改对后续的线程可见。

## 读锁

读锁为一个可重入的共享锁，它能够被多个线程同时持有，在没有其他写线程访问时，读锁总是或获取成功。

### 读锁的获取

读锁的获取可以通过ReadLock的lock()方法：

```java
public void lock() {
    sync.acquireShared(1);
}
```

Sync的acquireShared(int arg)定义在AQS中：

```java
public final void acquireShared(int arg) {
    if (tryAcquireShared(arg) < 0)
        doAcquireShared(arg);
}
```

tryAcqurireShared(int arg)尝试获取读同步状态，该方法主要用于获取共享式同步状态，获取成功返回 >= 0的返回结果，否则返回 < 0 的返回结果。

```java
protected final int tryAcquireShared(int unused) {
    //当前线程
    Thread current = Thread.currentThread();
  	// 获取状态值
    int c = getState();
    //exclusiveCount(c)计算写锁
    //如果存在写锁，且锁的持有者不是当前线程，直接返回-1
    //存在锁降级问题
    if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
        return -1;
    //读锁
    int r = sharedCount(c);

    /*
     * readerShouldBlock():读锁是否需要等待（公平锁原则）
     * r < MAX_COUNT：持有线程小于最大数（65535）
     * compareAndSetState(c, c + SHARED_UNIT)：设置读取锁状态
     */
    if (!readerShouldBlock() && r < MAX_COUNT && compareAndSetState(c, c + SHARED_UNIT)) {
        /*
         * holdCount 就是锁降级的关键 (后续着重讲解)
         */
        if (r == 0) {
            firstReader = current;
            firstReaderHoldCount = 1;
        } else if (firstReader == current) {
            firstReaderHoldCount++;
        } else {
            HoldCounter rh = cachedHoldCounter;
            if (rh == null || rh.tid != getThreadId(current))
                cachedHoldCounter = rh = readHolds.get();
            else if (rh.count == 0)
                readHolds.set(rh);
            rh.count++;
        }
        return 1;
    }
    return fullTryAcquireShared(current);
}
```

读锁获取的过程相对于独占锁而言会稍微复杂下，整个过程如下：

1. 因为存在锁降级情况，如果存在写锁且锁的持有者不是当前线程则直接返回失败，否则继续
2. 依据公平性原则，判断读锁是否需要阻塞，读锁持有线程数小于最大值（65535），且设置锁状态成功，执行以下代码（对于HoldCounter下面再阐述），并返回1。如果不满足改条件，执行fullTryAcquireShared()。

```java
final int fullTryAcquireShared(Thread current) {
    HoldCounter rh = null;
    for (;;) {
        int c = getState();
        //锁降级
        if (exclusiveCount(c) != 0) {
            if (getExclusiveOwnerThread() != current)
                return -1;
        }
        //读锁需要阻塞
        else if (readerShouldBlock()) {
            //列头为当前线程
            if (firstReader == current) {
            }
            //HoldCounter后面讲解
            else {
                if (rh == null) {
                    rh = cachedHoldCounter;
                    if (rh == null || rh.tid != getThreadId(current)) {
                        rh = readHolds.get();
                        if (rh.count == 0) readHolds.remove();
                    }
                }
                if (rh.count == 0) return -1;
            }
        }
        //读锁超出最大范围
        if (sharedCount(c) == MAX_COUNT) throw new Error("Maximum lock count exceeded");
        //CAS设置读锁成功
        if (compareAndSetState(c, c + SHARED_UNIT)) {
            //如果是第1次获取“读取锁”，则更新firstReader和firstReaderHoldCount
            if (sharedCount(c) == 0) {
                firstReader = current;
                firstReaderHoldCount = 1;
            }
            //如果想要获取锁的线程(current)是第1个获取锁(firstReader)的线程,则将firstReaderHoldCount+1
            else if (firstReader == current) {
                firstReaderHoldCount++;
            } else {
                if (rh == null)
                    rh = cachedHoldCounter;
                if (rh == null || rh.tid != getThreadId(current))
                    rh = readHolds.get();
                else if (rh.count == 0)
                    readHolds.set(rh);
                //更新线程的获取“读取锁”的共享计数
                rh.count++;
                cachedHoldCounter = rh; // cache for release
            }
            return 1;
        }
    }
}
```

fullTryAcquireShared(Thread current)会根据“是否需要阻塞等待”，“读取锁的共享计数是否超过限制”等等进行处理。如果不需要阻塞等待，并且锁的共享计数没有超过限制，则通过CAS尝试获取锁，并返回1

### 读锁的释放

```java
public void unlock() {
    sync.releaseShared(1);
}
```

unlcok()方法内部使用Sync的releaseShared(int arg)方法，该方法定义在AQS中：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

调用tryReleaseShared(int arg)尝试释放读锁，该方法定义在读写锁的Sync内部类中：

```java
protected final boolean tryReleaseShared(int unused) {
    Thread current = Thread.currentThread();
    //如果想要释放锁的线程为第一个获取锁的线程
    if (firstReader == current) {
        //仅获取了一次，则需要将firstReader 设置null，否则 firstReaderHoldCount - 1
        if (firstReaderHoldCount == 1) firstReader = null;
        else firstReaderHoldCount--;
    }
    //获取rh对象，并更新“当前线程获取锁的信息”
    else {
        HoldCounter rh = cachedHoldCounter;
        if (rh == null || rh.tid != getThreadId(current)) rh = readHolds.get();
      
        int count = rh.count;
        if (count <= 1) {
            readHolds.remove();
            if (count <= 0) throw unmatchedUnlockException();
        }
        --rh.count;
    }
    //CAS更新同步状态
    for (;;) {
        int c = getState();
        int nextc = c - SHARED_UNIT;
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

## HoldCounter

在读锁获取锁和释放锁的过程中，我们一直都可以看到一个变量rh （HoldCounter ），该变量在读锁中扮演着非常重要的作用。 我们了解读锁的内在机制其实就是一个共享锁，为了更好理解HoldCounter ，我们暂且认为它不是一个锁的概率，而相当于一个计数器。一次共享锁的操作就相当于在该计数器的操作。

**获取共享锁，则该计数器 + 1，释放共享锁，该计数器 - 1。**

只有当线程获取共享锁后才能对共享锁进行释放、重入操作。所以HoldCounter的作用就是当前线程持有共享锁的数量，这个数量必须要与线程绑定在一起，否则操作其他线程锁就会抛出异常。我们先看HoldCounter的定义：

```java
static final class HoldCounter {
    int count = 0;
    final long tid = getThreadId(Thread.currentThread());
}
```

HoldCounter 定义非常简单，就是一个计数器count 和线程 id tid 两个变量。按照这个意思我们看到HoldCounter 是需要和某给线程进行绑定了，我们知道如果要将一个对象和线程绑定仅仅有tid是不够的，而且从上面的代码我们可以看到HoldCounter 仅仅只是记录了tid，根本起不到绑定线程的作用。那么怎么实现呢？答案是ThreadLocal，定义如下：

```java
static final class ThreadLocalHoldCounter extends ThreadLocal<HoldCounter> {
    public HoldCounter initialValue() {
        return new HoldCounter();
    }
}
```

通过上面代码HoldCounter就可以与线程进行绑定了。

HoldCounter应该就是绑定线程上的一个计数器，而ThradLocalHoldCounter则是线程绑定的ThreadLocal。

从上面我们可以看到ThreadLocal将HoldCounter绑定到当前线程上，同时HoldCounter也持有线程Id，这样在释放锁的时候才能知道ReadWriteLock里面缓存的上一个读取线程（cachedHoldCounter）是否是当前线程。

这样做的好处是可以减少ThreadLocal.get()的次数，因为这也是一个耗时操作。需要说明的是这样HoldCounter绑定线程id而不绑定线程对象的原因是避免HoldCounter和ThreadLocal互相绑定而GC难以释放它们（尽管GC能够智能的发现这种引用而回收它们，但是这需要一定的代价），所以其实这样做只是为了帮助GC快速回收对象而已。 看到这里我们明白了HoldCounter作用了，我们在看一个获取读锁的代码段：

```java
//如果获取读锁的线程为第一次获取读锁的线程，则firstReaderHoldCount重入数 + 1
else if (firstReader == current) {
    firstReaderHoldCount++;
} else {
    //非firstReader计数
    if (rh == null)
        rh = cachedHoldCounter;
    //rh == null 或者 rh.tid != current.getId()，需要获取rh
    if (rh == null || rh.tid != getThreadId(current))
        rh = readHolds.get();
        //加入到readHolds中
    else if (rh.count == 0)
        readHolds.set(rh);
    //计数+1
    rh.count++;
    cachedHoldCounter = rh; // cache for release
}
```

这段代码涉及了几个变量：firstReader 、firstReaderHoldCount、cachedHoldCounter 。我们先理清楚这几个变量：

```java
private transient Thread firstReader = null;
private transient int firstReaderHoldCount;
private transient HoldCounter cachedHoldCounter;
```

firstReader 看名字就明白了为第一个获取读锁的线程，firstReaderHoldCount为第一个获取读锁的重入数，cachedHoldCounter为HoldCounter的缓存。 

为何要引入firstRead、firstReaderHoldCount?

这是为了一个效率问题，firstReader是不会放入到readHolds中的，如果读锁仅有一个的情况下就会避免查找readHolds。

## 锁降级

读写锁有一个特性就是锁降级，锁降级就意味着写锁是可以降级为读锁的，但是需要遵循先获取写锁、获取读锁在释放写锁的次序。注意如果当前线程先获取写锁，然后释放写锁，再获取读锁这个过程不能称之为锁降级，锁降级一定要遵循那个次序。 在获取读锁的方法tryAcquireShared(int unused)中，有一段代码就是来判读锁降级的：

```java
int c = getState();
//exclusiveCount(c)计算写锁
//如果存在写锁，且锁的持有者不是当前线程，直接返回-1
//存在锁降级问题
if (exclusiveCount(c) != 0 && getExclusiveOwnerThread() != current)
    return -1;
//读锁
int r = sharedCount(c);
```

锁降级中读锁的获取释放为必要？肯定是必要的。试想，假如当前线程A不获取读锁而是直接释放了写锁，这个时候另外一个线程B获取了写锁，那么这个线程B对数据的修改是不会对当前线程A可见的。如果获取了读锁，则线程B在获取写锁过程中判断如果有读锁还没有释放则会被阻塞，只有当前线程A释放读锁后，线程B才会获取写锁成功。

## Condition

> 在没有Lock之前，我们使用synchronized来控制同步，配合Object的wait()、notify()系列方法可以实现等待/通知模式。在Java SE5后，Java提供了Lock接口，相对于Synchronized而言，Lock提供了条件Condition，对线程的等待、唤醒操作更加详细和灵活。下图是Condition与Object的监视器方法的对比（摘自《Java并发编程的艺术》）：
>
> ![](http://image.tinx.top/20211129112804.png)

Condition提供了一系列的方法来对阻塞和唤醒线程：

1. **await()** ：造成当前线程在接到信号或被中断之前一直处于等待状态。
2. **await(long time, TimeUnit unit)** ：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。
3. **awaitNanos(long nanosTimeout)** ：造成当前线程在接到信号、被中断或到达指定等待时间之前一直处于等待状态。返回值表示剩余时间，如果在nanosTimesout之前唤醒，那么返回值 = nanosTimeout - 消耗时间，如果返回值 <= 0 ,则可以认定它已经超时了。
4. **awaitUninterruptibly()** ：造成当前线程在接到信号之前一直处于等待状态。**【注意：该方法对中断不敏感】**
5. **awaitUntil(Date deadline)** ：造成当前线程在接到信号、被中断或到达指定最后期限之前一直处于等待状态。如果没有到指定时间就被通知，则返回true，否则表示到了指定时间，返回返回false。
6. **signal()** ：唤醒一个等待线程。该线程从等待方法返回前必须获得与Condition相关的锁。
7. **signal()All** ：唤醒所有等待线程。能够从等待方法返回的线程必须获得与Condition相关的锁。

Condition是一种广义上的条件队列。他为线程提供了一种更为灵活的等待/通知模式，线程在调用await方法后执行挂起操作，直到线程等待的某个条件为真时才会被唤醒。Condition必须要配合锁一起使用，因为对共享状态变量的访问发生在多线程环境下。一个Condition的实例必须与一个Lock绑定，因此Condition一般都是作为Lock的内部实现。

### Condition实现

获取一个Condition必须要通过Lock的newCondition()方法。该方法定义在接口Lock下面，返回的结果是绑定到此 Lock 实例的新 Condition 实例。Condition为一个接口，其下仅有一个实现类ConditionObject，由于Condition的操作需要获取相关的锁，而AQS则是同步锁的实现基础，所以ConditionObject则定义为AQS的内部类。定义如下：

```java
public class ConditionObject implements Condition, java.io.Serializable {
}
```

#### 等待队列

每个Condition对象都包含着一个FIFO队列，该队列是Condition对象通知/等待功能的关键。在队列中每一个节点都包含着一个线程引用，该线程就是在该Condition对象上等待的线程。我们看Condition的定义就明白了：

```java
public class ConditionObject implements Condition, java.io.Serializable {
    //头节点
    private transient Node firstWaiter;
    //尾节点
    private transient Node lastWaiter;

    public ConditionObject() {
    }

    /** 省略方法 **/
}
```

从上面代码可以看出Condition拥有首节点（firstWaiter），尾节点（lastWaiter）。当前线程调用await()方法，将会以当前线程构造成一个节点（Node），并将节点加入到该队列的尾部。结构如下：

![](http://image.tinx.top/20211129113349.png)

Node里面包含了当前线程的引用。Node定义与AQS的CLH同步队列的节点使用的都是同一个类（AbstractQueuedSynchronized.Node静态内部类）。

Condition的队列结构比CLH同步队列的结构简单些，新增过程较为简单只需要将原尾节点的nextWaiter指向新增节点，然后更新lastWaiter即可。

##### 等待

调用Condition的await()方法会使当前线程进入等待状态，同时会加入到Condition等待队列同时释放锁。当从await()方法返回时，当前线程一定是获取了Condition相关连的锁。

```java
public final void await() throws InterruptedException {
    // 当前线程中断
    if (Thread.interrupted()) throw new InterruptedException();
    //当前线程加入等待队列
    Node node = addConditionWaiter();
    //释放锁
    long savedState = fullyRelease(node);
    int interruptMode = 0;
    /**
     * 检测此节点的线程是否在同步队上，如果不在，则说明该线程还不具备竞争锁的资格，则继续等待
     * 直到检测到此节点在同步队列上
     */
    while (!isOnSyncQueue(node)) {
        //线程挂起
        LockSupport.park(this);
        //如果已经中断了，则退出
        if ((interruptMode = checkInterruptWhileWaiting(node)) != 0)
            break;
    }
    //竞争同步状态
    if (acquireQueued(node, savedState) && interruptMode != THROW_IE)
        interruptMode = REINTERRUPT;
    //清理下条件队列中的不是在等待条件的节点
    if (node.nextWaiter != null)
        unlinkCancelledWaiters();
    if (interruptMode != 0)
        reportInterruptAfterWait(interruptMode);
}
```

此段代码的逻辑是：首先将当前线程新建一个节点同时加入到条件队列中，然后释放当前线程持有的同步状态。然后则是不断检测该节点代表的线程释放出现在CLH同步队列中（收到signal信号之后就会在AQS队列中检测到），如果不存在则一直挂起，否则参与竞争同步状态。

加入条件队列（addConditionWaiter()）源码如下：

```java
private Node addConditionWaiter() {
    Node t = lastWaiter;    //尾节点
    //Node的节点状态如果不为CONDITION，则表示该节点不处于等待状态，需要清除节点
    if (t != null && t.waitStatus != Node.CONDITION) {
        //清除条件队列中所有状态不为Condition的节点
        unlinkCancelledWaiters();
        t = lastWaiter;
    }
    //当前线程新建节点，状态CONDITION
    Node node = new Node(Thread.currentThread(), Node.CONDITION);
    /**
     * 将该节点加入到条件队列中最后一个位置
     */
    if (t == null)
        firstWaiter = node;
    else
        t.nextWaiter = node;
    lastWaiter = node;
    return node;
}
```

该方法主要是将当前线程加入到Condition条件队列中。当然在加入到尾节点之前会清楚所有状态不为Condition的节点。

fullyRelease(Node node)，负责释放该线程持有的锁。

```java
final long fullyRelease(Node node) {
    boolean failed = true;
    try {
        //节点状态--其实就是持有锁的数量
        long savedState = getState();
        //释放锁
        if (release(savedState)) {
            failed = false;
            return savedState;
        } else  throw new IllegalMonitorStateException();
    } finally {
        if (failed)
            node.waitStatus = Node.CANCELLED;
    }
}
```

isOnSyncQueue(Node node)：如果一个节点刚开始在条件队列上，现在在同步队列上获取锁则返回true

```java
final boolean isOnSyncQueue(Node node) {
    //状态为Condition，获取前驱节点为null，返回false
    if (node.waitStatus == Node.CONDITION || node.prev == null)
        return false;
    //后继节点不为null，肯定在CLH同步队列中
    if (node.next != null)
        return true;

    return findNodeFromTail(node);
}
```

unlinkCancelledWaiters()：负责将条件队列中状态不为Condition的节点删除

```java
private void unlinkCancelledWaiters() {
    Node t = firstWaiter;
    Node trail = null;
    while (t != null) {
        Node next = t.nextWaiter;
        if (t.waitStatus != Node.CONDITION) {
            t.nextWaiter = null;
            if (trail == null)
                firstWaiter = next;
            else
                trail.nextWaiter = next;
            if (next == null)
                lastWaiter = trail;
        }
        else
            trail = t;
        t = next;
    }
}
```

##### 通知

调用Condition的signal()方法，将会唤醒在等待队列中等待最长时间的节点（条件队列里的首节点），在唤醒节点前，会将节点移到CLH同步队列中。

```java
public final void signal() {
    //检测当前线程是否拥有锁
    if (!isHeldExclusively())
        throw new IllegalMonitorStateException();
    //头节点，唤醒条件队列中的第一个节点
    Node first = firstWaiter;
    if (first != null)
        doSignal(first);    //唤醒
}
```

该方法首先会判断当前线程是否已经获得了锁，这是前置条件。然后唤醒条件队列中的头节点。

doSignal(Node first)：唤醒头节点

```java
private void doSignal(Node first) {
    do {
        //修改头结点，完成旧头结点的移出工作
        if ( (firstWaiter = first.nextWaiter) == null) lastWaiter = null;
        first.nextWaiter = null;
    } while (!transferForSignal(first) && (first = firstWaiter) != null);
}
```

doSignal(Node first)主要是做两件事：1.修改头节点，2.调用transferForSignal(Node first) 方法将节点移动到CLH同步队列中。transferForSignal(Node first)源码如下：

```java
final boolean transferForSignal(Node node) {
    //将该节点从状态CONDITION改变为初始状态0,
    if (!compareAndSetWaitStatus(node, Node.CONDITION, 0))
        return false;

    //将节点加入到syn队列中去，返回的是syn队列中node节点前面的一个节点
    Node p = enq(node);
    int ws = p.waitStatus;
    //如果结点p的状态为cancel 或者修改waitStatus失败，则直接唤醒
    if (ws > 0 || !compareAndSetWaitStatus(p, ws, Node.SIGNAL))
        LockSupport.unpark(node.thread);
    return true;
}
```

整个通知的流程如下：

1. 判断当前线程是否已经获取了锁，如果没有获取则直接抛出异常，因为获取锁为通知的前置条件。
2. 如果线程已经获取了锁，则将唤醒条件队列的首节点
3. 唤醒首节点是先将条件队列中的头节点移出，然后调用AQS的enq(Node node)方法将其安全地移到CLH同步队列中
4. 最后判断如果该节点的同步状态是否为Cancel，或者修改状态为Signal失败时，则直接调用LockSupport唤醒该节点的线程。

##### 总结

一个线程获取锁后，通过调用Condition的await()方法，会将当前线程先加入到条件队列中，然后释放锁，最后通过isOnSyncQueue(Node node)方法不断自检看节点是否已经在CLH同步队列了，如果是则尝试获取锁，否则一直挂起。当线程调用signal()方法后，程序首先检查当前线程是否获取了锁，然后通过doSignal(Node first)方法唤醒CLH同步队列的首节点。被唤醒的线程，将从await()方法中的while循环中退出来，然后调用acquireQueued()方法竞争同步状态。

##### 小例子

```java
public class ConditionTest {
    private LinkedList<String> buffer;    //容器
    private int maxSize ;           //容器最大
    private Lock lock;
    private Condition fullCondition;
    private Condition notFullCondition;
		
  	// 构造方法，初始化一些参数
    ConditionTest(int maxSize){
        this.maxSize = maxSize;
        buffer = new LinkedList<String>();
        lock = new ReentrantLock();
        fullCondition = lock.newCondition();
        notFullCondition = lock.newCondition();
    }

    public void set(String string) throws InterruptedException {
        lock.lock();    //获取锁
        try {
            while (maxSize == buffer.size()){
                notFullCondition.await();       //满了，添加的线程进入等待状态
            }

            buffer.add(string);
            fullCondition.signal();
        } finally {
            lock.unlock();      //记得释放锁
        }
    }

    public String get() throws InterruptedException {
        String string;
        lock.lock();
        try {
            while (buffer.size() == 0){
                fullCondition.await();
            }
            string = buffer.poll();
            notFullCondition.signal();
        } finally {
            lock.unlock();
        }
        return string;
    }
}
```



# JUC并发工具类及小例子

## CyclicBarrier

> CyclicBarrier，一个同步辅助类，在API中是这么介绍的： 它允许一组线程互相等待，直到到达某个公共屏障点 (common barrier point)。在涉及一组固定大小的线程的程序中，这些线程必须不时地互相等待，此时 CyclicBarrier 很有用。因为该 barrier 在释放等待线程后可以重用，所以称它为循环 的 barrier。 通俗点讲就是：让一组线程到达一个屏障时被阻塞，直到最后一个线程到达屏障时，屏障才会开门，所有被屏障拦截的线程才会继续干活。

### 架构图

![](http://image.tinx.top/20211129134012.png)

通过上图我们可以看到CyclicBarrier的内部是使用重入锁ReentrantLock和Condition。它有两个构造函数：

- CyclicBarrier(int parties)：创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，但它不会在启动 barrier 时执行预定义的操作。
- CyclicBarrier(int parties, Runnable barrierAction) ：创建一个新的 CyclicBarrier，它将在给定数量的参与者（线程）处于等待状态时启动，并在启动 barrier 时执行给定的屏障操作，该操作由最后一个进入 barrier 的线程执行。

parties表示拦截线程的数量。 barrierAction 为CyclicBarrier接收的Runnable命令，用于在线程到达屏障时，优先执行barrierAction ，用于处理更加复杂的业务场景。

```java
public CyclicBarrier(int parties, Runnable barrierAction) {
    if (parties <= 0) throw new IllegalArgumentException();
    this.parties = parties;
    this.count = parties;
    this.barrierCommand = barrierAction;
}

public CyclicBarrier(int parties) {
    this(parties, null);
}
```

在CyclicBarrier中最重要的方法莫过于await()方法，在所有参与者都已经在此 barrier 上调用 await 方法之前，将一直等待。如下：

```java
public int await() throws InterruptedException, BrokenBarrierException {
    try {
        return dowait(false, 0L);//不超时等待
    } catch (TimeoutException toe) {
        throw new Error(toe); // cannot happen
    }
}
```

await()方法内部调用dowait(boolean timed, long nanos)方法：

```java
private int dowait(boolean timed, long nanos) throws InterruptedException, BrokenBarrierException, TimeoutException {
    //获取锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        //分代
        final Generation g = generation;

        //当前generation“已损坏”，抛出BrokenBarrierException异常
        //抛出该异常一般都是某个线程在等待某个处于“断开”状态的CyclicBarrie
        if (g.broken) throw new BrokenBarrierException();

        //如果线程中断，终止CyclicBarrier
        if (Thread.interrupted()) {
            breakBarrier();
            throw new InterruptedException();
        }

        //进来一个线程 count - 1
        int index = --count;
        //count == 0 表示所有线程均已到位，触发Runnable任务
        if (index == 0) {  // tripped
            boolean ranAction = false;
            try {
                final Runnable command = barrierCommand;
                //触发任务
                if (command != null)
                    command.run();
                ranAction = true;
                //唤醒所有等待线程，并更新generation
                nextGeneration();
                return 0;
            } finally {
                if (!ranAction)
                    breakBarrier();
            }
        }


        for (;;) {
            try {
                //如果不是超时等待，则调用Condition.await()方法等待
                if (!timed)
                    trip.await();
                else if (nanos > 0L)
                    //超时等待，调用Condition.awaitNanos()方法等待
                    nanos = trip.awaitNanos(nanos);
            } catch (InterruptedException ie) {
                if (g == generation && ! g.broken) {
                    breakBarrier();
                    throw ie;
                } else {
                    // We're about to finish waiting even if we had not
                    // been interrupted, so this interrupt is deemed to
                    // "belong" to subsequent execution.
                    Thread.currentThread().interrupt();
                }
            }

            if (g.broken)
                throw new BrokenBarrierException();

            //generation已经更新，返回index
            if (g != generation)
                return index;

            //“超时等待”，并且时间已到,终止CyclicBarrier，并抛出异常
            if (timed && nanos <= 0L) {
                breakBarrier();
                throw new TimeoutException();
            }
        }
    } finally {
        //释放锁
        lock.unlock();
    }
}
```

其实await()的处理逻辑还是比较简单的：如果该线程不是到达的最后一个线程，则他会一直处于等待状态，除非发生以下情况：

1. 最后一个线程到达，即index == 0
2. 超出了指定时间（超时等待）
3. 其他的某个线程中断当前线程
4. 其他的某个线程中断另一个等待的线程
5. 其他的某个线程在等待barrier超时
6. 其他的某个线程在此barrier调用reset()方法。reset()方法用于将屏障重置为初始状态。

在上面的源代码中，我们可能需要注意Generation 对象，在上述代码中我们总是可以看到抛出BrokenBarrierException异常，那么什么时候抛出异常呢？如果一个线程处于等待状态时，如果其他线程调用reset()，或者调用的barrier原本就是被损坏的，则抛出BrokenBarrierException异常。同时，任何线程在等待时被中断了，则其他所有线程都将抛出BrokenBarrierException异常，并将barrier置于损坏状态。 同时，Generation描述着CyclicBarrier的更显换代。在CyclicBarrier中，同一批线程属于同一代。当有parties个线程到达barrier，generation就会被更新换代。其中broken标识该当前CyclicBarrier是否已经处于中断状态。

```java
private static class Generation {
    boolean broken = false;
}
```

默认barrier是没有损坏的。 当barrier损坏了或者有一个线程中断了，则通过breakBarrier()来终止所有的线程：

```java
private void breakBarrier() {
    generation.broken = true;
    count = parties;
    trip.signalAll();
}
```

在breakBarrier()中除了将broken设置为true，还会调用signalAll将在CyclicBarrier处于等待状态的线程全部唤醒。 当所有线程都已经到达barrier处（index == 0），则会通过nextGeneration()进行更新换地操作，在这个步骤中，做了三件事：唤醒所有线程，重置count，generation。

```java
private void nextGeneration() {
    trip.signalAll();
    count = parties;
    generation = new Generation();
}
```

CyclicBarrier同时也提供了await(long timeout, TimeUnit unit) 方法来做超时控制，内部还是通过调用doawait()实现的。

### 应用场景

CyclicBarrier试用与多线程结果合并的操作，用于多线程计算数据，最后合并计算结果的应用场景。比如我们需要统计多个Excel中的数据，然后等到一个总结果。我们可以通过多线程处理每一个Excel，执行完成后得到相应的结果，最后通过barrierAction来计算这些线程的计算结果，得到所有Excel的总和。

### 小例子

```java
public class CyclicBarrierTest {

    private static CyclicBarrier cyclicBarrier;

    static class Employee extends Thread{

        @Override
        public void run() {
            System.out.println("职员：" + Thread.currentThread().getName() + "到达了会议现场");
            try {
                cyclicBarrier.await();
                System.out.println(Thread.currentThread().getName() + "发表了自己的观点");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } catch (BrokenBarrierException e) {
                e.printStackTrace();
            }
        }
    }

    public static void main(String[] args) {
        int count = 5;
        Random random = new Random();
        cyclicBarrier = new CyclicBarrier(count, () -> {
            System.out.println("人已经到齐" + count + "开始开会！");
        });
        for (int i = 0; i < count; i++) {
            new Employee().start();
            try {
                Thread.sleep(random.nextInt(3000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
        }
    }
}
```

![](http://image.tinx.top/20211129135604.png)

## CountDownLatch

> CyclicBarrier所描述的是“允许一组线程互相等待，直到到达某个公共屏障点，才会进行后续任务"，而CountDownLatch所描述的是”在完成一组正在其他线程中执行的操作之前，它允许一个或多个线程一直等待“。在API中是这样描述的：
>
> > 用给定的计数 初始化 CountDownLatch。由于调用了 countDown() 方法，所以在当前计数到达零之前，await 方法会一直受阻塞。之后，会释放所有等待的线程，await 的所有后续调用都将立即返回。这种现象只出现一次——计数无法被重置。如果需要重置计数，请考虑使用 CyclicBarrier。

CountDownLatch是通过一个计数器来实现的，当我们在new 一个CountDownLatch对象的时候需要带入该计数器值，该值就表示了线程的数量。每当一个线程完成自己的任务后，计数器的值就会减1。当计数器的值变为0时，就表示所有的线程均已经完成了任务，然后就可以恢复等待的线程继续执行了。

虽然，CountDownlatch与CyclicBarrier有那么点相似，但是他们还是存在一些区别的：

1. CountDownLatch的作用是允许1或N个线程等待其他线程完成执行；而CyclicBarrier则是允许N个线程相互等待
2. CountDownLatch的计数器无法被重置；CyclicBarrier的计数器可以被重置后使用，因此它被称为是循环的barrier

<img src="http://image.tinx.top/20211129141753.png" style="zoom:50%;" />

### 架构图

![](http://image.tinx.top/20211129141826.png)

通过上面的结构图我们可以看到，CountDownLatch内部依赖Sync实现，而Sync继承AQS。CountDownLatch仅提供了一个构造方法：

CountDownLatch(int count) ： 构造一个用给定计数初始化的 CountDownLatch

```java
public CountDownLatch(int count) {
    if (count < 0) throw new IllegalArgumentException("count < 0");
    this.sync = new Sync(count);
}
```

sync为CountDownLatch的一个内部类，其定义如下：

```java
private static final class Sync extends AbstractQueuedSynchronizer {
  
    Sync(int count) {
        setState(count);
    }

    //获取同步状态
    int getCount() {
        return getState();
    }

    //获取同步状态
    protected int tryAcquireShared(int acquires) {
        return (getState() == 0) ? 1 : -1;
    }

    //释放同步状态
    protected boolean tryReleaseShared(int releases) {
        for (;;) {
            int c = getState();
            if (c == 0)
                return false;
            int nextc = c-1;
            if (compareAndSetState(c, nextc))
                return nextc == 0;
        }
    }
}
```

通过这个内部类Sync我们可以清楚地看到CountDownLatch是采用共享锁来实现的。

### await()

CountDownLatch提供await()方法来使当前线程在锁存器倒计数至零之前一直等待，除非线程被中断，定义如下：

```java
public void await() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

await其内部使用AQS的acquireSharedInterruptibly(int arg)：

```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted())
        throw new InterruptedException();
    if (tryAcquireShared(arg) < 0)
        doAcquireSharedInterruptibly(arg);
}
```

在内部类Sync中重写了tryAcquireShared(int arg)方法：

```java
protected int tryAcquireShared(int acquires) {
    return (getState() == 0) ? 1 : -1;
}
```

getState()获取同步状态，其值等于计数器的值，从这里我们可以看到如果计数器值不等于0，则会调用doAcquireSharedInterruptibly(int arg)，该方法为一个自旋方法会尝试一直去获取同步状态：

```java
private void doAcquireSharedInterruptibly(int arg) throws InterruptedException {
    final Node node = addWaiter(Node.SHARED);
    boolean failed = true;
    try {
        for (;;) {
            final Node p = node.predecessor();
            if (p == head) {
                /**
                 * 对于CountDownLatch而言，如果计数器值不等于0，那么r 会一直小于0
                 */
                int r = tryAcquireShared(arg);
                if (r >= 0) {
                    setHeadAndPropagate(node, r);
                    p.next = null; // help GC
                    failed = false;
                    return;
                }
            }
            //等待
            if (shouldParkAfterFailedAcquire(p, node) && parkAndCheckInterrupt())
                throw new InterruptedException();
        }
    } finally {
        if (failed)
            cancelAcquire(node);
    }
}
```

### countDown()

CountDownLatch提供countDown() 方法递减锁存器的计数，如果计数到达零，则释放所有等待的线程。

```java
public void countDown() {
    sync.releaseShared(1);
}
```

内部调用AQS的releaseShared(int arg)方法来释放共享锁同步状态：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

tryReleaseShared(int arg)方法被CountDownLatch的内部类Sync重写：

```java
protected boolean tryReleaseShared(int releases) {
    for (;;) {
        //获取锁状态
        int c = getState();
        //c == 0 直接返回，释放锁成功
        if (c == 0)
            return false;
        //计算新“锁计数器”
        int nextc = c-1;
        //更新锁状态（计数器）
        if (compareAndSetState(c, nextc))
            return nextc == 0;
    }
}
```

### 总结

CountDownLatch内部通过共享锁实现。在创建CountDownLatch实例时，需要传递一个int型的参数：count，该参数为计数器的初始值，也可以理解为该共享锁可以获取的总次数。

当某个线程调用await()方法，程序首先判断count的值是否为0，如果不会0的话则会一直等待直到为0为止。

当其他线程调用countDown()方法时，则执行释放共享锁状态，使count值 - 1。

当在创建CountDownLatch时初始化的count参数，必须要有count线程调用countDown方法才会使计数器count等于0，锁才会释放，前面等待的线程才会继续运行。***注意CountDownLatch不能回滚重置。***

### 小例子

```java
public class CountDownLatchTest {

    private static CountDownLatch countDownLatch = new CountDownLatch(5);

    /**
     * Boss线程，等待员工到达开会
     */
    static class Boss extends Thread{
        @Override
        public void run() {
            System.out.println("Boss在会议室等待，总共有" + countDownLatch.getCount() + "个人开会...");
            try {
                //Boss等待
                countDownLatch.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }

            System.out.println("所有人都已经到齐了，开会吧...");
        }
    }

    //员工到达会议室
    static class Empleoyee  extends Thread{
        @Override
        public void run() {
            try {
                Thread.sleep(new Random().nextInt(3000));
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            System.out.println(Thread.currentThread().getName() + "，到达会议室....");
            //员工到达会议室 count - 1
            countDownLatch.countDown();
        }
    }

    public static void main(String[] args){
        //Boss线程启动
        new Boss().start();

        for(int i = 0, j = Integer.parseInt(String.valueOf(countDownLatch.getCount())); i < j ; i++){
            new Empleoyee().start();
        }
    }
}
```

![](http://image.tinx.top/20211129142827.png)

## Semaphore

> 信号量Semaphore是一个控制访问多个共享资源的计数器，和CountDownLatch一样，其本质上是一个“共享锁”。
>
> Semaphore，在API是这么介绍的：
>
> 一个计数信号量。从概念上讲，信号量维护了一个许可集。如有必要，在许可可用前会阻塞每一个 acquire()，然后再获取该许可。每个 release() 添加一个许可，从而可能释放一个正在阻塞的获取者。但是，不使用实际的许可对象，Semaphore 只对可用许可的号码进行计数，并采取相应的行动。
>
> Semaphore 通常用于限制可以访问某些资源（物理或逻辑的）的线程数目。

<img src="http://image.tinx.top/20211129143310.png" style="zoom:50%;float:left;" />

从上图可以看出Semaphore内部包含公平锁（FairSync）和非公平锁（NonfairSync），继承内部类Sync，其中Sync继承AQS（再一次阐述AQS的重要性）。

Semaphore提供了两个构造函数：

1. Semaphore(int permits) ：创建具有给定的许可数和非公平的公平设置的 Semaphore。
2. Semaphore(int permits, boolean fair) ：创建具有给定的许可数和给定的公平设置的 Semaphore。

```java
public Semaphore(int permits) {
    sync = new NonfairSync(permits);
}

public Semaphore(int permits, boolean fair) {
    sync = fair ? new FairSync(permits) : new NonfairSync(permits);
}
```

Semaphore默认选择非公平锁。

当信号量Semaphore = 1 时，它可以当作互斥锁使用。其中0、1就相当于它的状态，当=1时表示其他线程可以获取，当=0时，排他，即其他线程必须要等待。

### 信号量获取

Semaphore提供了acquire()方法来获取一个许可。

```java
public void acquire() throws InterruptedException {
    sync.acquireSharedInterruptibly(1);
}
```

内部调用AQS的acquireSharedInterruptibly(int arg)，该方法以共享模式获取同步状态：

```java
public final void acquireSharedInterruptibly(int arg) throws InterruptedException {
    if (Thread.interrupted()) throw new InterruptedException();
    if (tryAcquireShared(arg) < 0) doAcquireSharedInterruptibly(arg);
}
```

在acquireSharedInterruptibly(int arg)中，tryAcquireShared(int arg)由子类来实现，对于Semaphore而言，如果我们选择非公平模式，则调用NonfairSync的tryAcquireShared(int arg)方法，否则调用FairSync的tryAcquireShared(int arg)方法。

**公平**

```java
protected int tryAcquireShared(int acquires) {
    for (;;) {
        //判断该线程是否位于CLH队列的列头
        if (hasQueuedPredecessors())
            return -1;
        //获取当前的信号量许可
        int available = getState();

        //设置“获得acquires个信号量许可之后，剩余的信号量许可数”
        int remaining = available - acquires;

        //CAS设置信号量
        if (remaining < 0 ||
                compareAndSetState(available, remaining))
            return remaining;
    }
}
```

**非公平**

对于非公平而言，因为它不需要判断当前线程是否位于CLH同步队列列头，所以相对而言会简单些。

```java
protected int tryAcquireShared(int acquires) {
    return nonfairTryAcquireShared(acquires);
}

final int nonfairTryAcquireShared(int acquires) {
    for (;;) {
        int available = getState();
        int remaining = available - acquires;
        if (remaining < 0 ||
            compareAndSetState(available, remaining))
            return remaining;
    }
}
```

### 信号量释放

获取了许可，当用完之后就需要释放，Semaphore提供release()来释放许可。

```java
public void release() {
    sync.releaseShared(1);
}
```

内部调用AQS的releaseShared(int arg)：

```java
public final boolean releaseShared(int arg) {
    if (tryReleaseShared(arg)) {
        doReleaseShared();
        return true;
    }
    return false;
}
```

releaseShared(int arg)调用Semaphore内部类Sync的tryReleaseShared(int arg)：

```java
protected final boolean tryReleaseShared(int releases) {
    for (;;) {
        int current = getState();
        //信号量的许可数 = 当前信号许可数 + 待释放的信号许可数
        int next = current + releases;
        if (next < current)  throw new Error("Maximum permit count exceeded");
        //设置可获取的信号许可数为next
        if (compareAndSetState(current, next))
            return true;
    }
}
```

### 小案例

```java
public class SemaphoreTest {


    static class Parking{
        //信号量
        private Semaphore semaphore;

        Parking(int count){
            semaphore = new Semaphore(count);
        }

        public void park(){
            try {
                //获取信号量
                semaphore.acquire();
                long time = new Random().nextInt(3000);
                System.out.println(Thread.currentThread().getName() + "进入停车场，停车" + (double)(time / 1000) + "秒..." );
                Thread.sleep(time);
                System.out.println(Thread.currentThread().getName() + "开出停车场...");
            } catch (InterruptedException e) {
                e.printStackTrace();
            } finally {
                semaphore.release();
            }
        }
    }

    static class Car extends Thread {
        Parking parking ;

        Car(Parking parking){
            this.parking = parking;
        }

        @Override
        public void run() {
            parking.park();     //进入停车场
        }
    }

    public static void main(String[] args){
        Parking parking = new Parking(3);

        for(int i = 0 ; i < 50 ; i++){
            new Car(parking).start();
        }
    }
}
```

<img src="http://image.tinx.top/20211129143701.png" style="zoom:50%;float:left;" />

## Exchanger

> Exchange是最简单的也是最复杂的，简单在于API非常简单，就一个构造方法和两个exchange()方法，最复杂在于它的实现是最复杂的（反正我是看晕了的）。 
>
> 在API是这么介绍的：可以在对中对元素进行配对和交换的线程的同步点。每个线程将条目上的某个方法呈现给 exchange 方法，与伙伴线程进行匹配，并且在返回时接收其伙伴的对象。Exchanger 可能被视为 SynchronousQueue 的双向形式。Exchanger 可能在应用程序（比如遗传算法和管道设计）中很有用。 Exchanger，它允许在并发任务之间交换数据。具体来说，Exchanger类允许在两个线程之间定义同步点。当两个线程都到达同步点时，他们交换数据结构，因此第一个线程的数据结构进入到第二个线程中，第二个线程的数据结构进入到第一个线程中。

### 源码分析

Exchanger算法的核心是通过一个可交换数据的slot，以及一个可以带有数据item的参与者。伪代码描述如下：

```java
for (;;) {
  if (slot is empty) {                       // offer
    place item in a Node;
    if (can CAS slot from empty to node) {
      wait for release;
      return matching item in node;
    }
  }
  else if (can CAS slot from node to empty) { // release
    get the item in node;
    set matching item in node;
    release waiting thread;
  }
  // else retry on CAS failure
}
```

Exchanger中定义了如下几个重要的成员变量：

```java
private final Participant participant;
private volatile Node[] arena;
private volatile Node slot;
```

participant的作用是为每个线程保留唯一的一个Node节点。 

slot为单个槽，arena为数组槽。他们都是Node类型。在这里可能会感觉到疑惑，slot作为Exchanger交换数据的场景，应该只需要一个就可以了啊？为何还多了一个Participant 和数组类型的arena呢？

​	一个slot交换场所原则上来说应该是可以的，但实际情况却不是如此，多个参与者使用同一个交换场所时，会存在严重伸缩性问题。既然单个交换场所存在问题，那么我们就安排多个，也就是数组arena。通过数组arena来安排不同的线程使用不同的slot来降低竞争问题，并且可以保证最终一定会成对交换数据。但是Exchanger不是一来就会生成arena数组来降低竞争，只有当产生竞争是才会生成arena数组。那么怎么将Node与当前线程绑定呢？

​	Participant ，Participant 的作用就是为每个线程保留唯一的一个Node节点，它继承ThreadLocal，同时在Node节点中记录在arena中的下标index。 Node定义如下：

```java
@sun.misc.Contended static final class Node {
    int index;              // arena的下标
    int bound;              // 上一次记录的Exchanger.bound
    int collides;           // 在当前bound下CAS失败的次数
    int hash;               // 伪随机数，用于自旋
    Object item;            // 这个线程的当前项，也就是需要交换的数据
    volatile Object match;  // 做releasing操作的线程传递的项
    volatile Thread parked; // 挂起时设置线程值，其他情况下为null
}
```

在Node定义中有两个变量值得思考：bound以及collides。前面提到了数组area是为了避免竞争而产生的，如果系统不存在竞争问题，那么完全没有必要开辟一个高效的arena来徒增系统的复杂性。首先通过单个slot的exchanger来交换数据，当探测到竞争时将安排不同的位置的slot来保存线程Node，并且可以确保没有slot会在同一个缓存行上。如何来判断会有竞争呢？CAS替换slot失败，如果失败，则通过记录冲突次数来扩展arena的尺寸，我们在记录冲突的过程中会跟踪“bound”的值，以及会重新计算冲突次数在bound的值被改变时。这里阐述可能有点儿模糊，不着急，我们先有这个概念，后面在arenaExchange中再次做详细阐述。 我们直接看exchange()方法

### exchange(V x)

**exchange(V x)**：等待另一个线程到达此交换点（除非当前线程被中断），然后将给定的对象传送给该线程，并接收该线程的对象。方法定义如下：

```java
public V exchange(V x) throws InterruptedException {
    Object v;
    Object item = (x == null) ? NULL_ITEM : x; // translate null args
    if ((arena != null || (v = slotExchange(item, false, 0L)) == null) &&
        ((Thread.interrupted() ||  (v = arenaExchange(item, false, 0L)) == null)))
        throw new InterruptedException();
    return (v == NULL_ITEM) ? null : (V)v;
}
```

这个方法比较好理解：arena为数组槽，如果为null，则执行slotExchange()方法，否则判断线程是否中断，如果中断值抛出InterruptedException异常，没有中断则执行arenaExchange()方法。

整套逻辑就是：

如果slotExchange(Object item, boolean timed, long ns)方法执行失败了就执行arenaExchange(Object item, boolean timed, long ns)方法，最后返回结果V。 NULL_ITEM 为一个空节点，其实就是一个Object对象而已，slotExchange()为单个slot交换。 **slotExchange(Object item, boolean timed, long ns)**

```java
private final Object slotExchange(Object item, boolean timed, long ns) {
    // 获取当前线程的节点 p
    Node p = participant.get();
    // 当前线程
    Thread t = Thread.currentThread();
    // 线程中断，直接返回
    if (t.isInterrupted())
        return null;
    // 自旋
    for (Node q;;) {
        //slot != null
        if ((q = slot) != null) {
            //尝试CAS替换
            if (U.compareAndSwapObject(this, SLOT, q, null)) {
                Object v = q.item;      // 当前线程的项，也就是交换的数据
                q.match = item;         // 做releasing操作的线程传递的项
                Thread w = q.parked;    // 挂起时设置线程值
                // 挂起线程不为null，线程挂起
                if (w != null)
                    U.unpark(w);
                return v;
            }
            //如果失败了，则创建arena
            //bound 则是上次Exchanger.bound
            if (NCPU > 1 && bound == 0 &&
                    U.compareAndSwapInt(this, BOUND, 0, SEQ))
                arena = new Node[(FULL + 2) << ASHIFT];
        }
        //如果arena != null，直接返回，进入arenaExchange逻辑处理
        else if (arena != null)
            return null;
        else {
            p.item = item;
            if (U.compareAndSwapObject(this, SLOT, null, p))
                break;
            p.item = null;
        }
    }

    /*
     * 等待 release
     * 进入spin+block模式
     */
    int h = p.hash;
    long end = timed ? System.nanoTime() + ns : 0L;
    int spins = (NCPU > 1) ? SPINS : 1;
    Object v;
    while ((v = p.match) == null) {
        if (spins > 0) {
            h ^= h << 1; h ^= h >>> 3; h ^= h << 10;
            if (h == 0)
                h = SPINS | (int)t.getId();
            else if (h < 0 && (--spins & ((SPINS >>> 1) - 1)) == 0)
                Thread.yield();
        }
        else if (slot != p)
            spins = SPINS;
        else if (!t.isInterrupted() && arena == null &&
                (!timed || (ns = end - System.nanoTime()) > 0L)) {
            U.putObject(t, BLOCKER, this);
            p.parked = t;
            if (slot == p)
                U.park(false, ns);
            p.parked = null;
            U.putObject(t, BLOCKER, null);
        }
        else if (U.compareAndSwapObject(this, SLOT, p, null)) {
            v = timed && ns <= 0L && !t.isInterrupted() ? TIMED_OUT : null;
            break;
        }
    }
    U.putOrderedObject(p, MATCH, null);
    p.item = null;
    p.hash = h;
    return v;
}
```

大体流程如下：

- 程序首先通过participant获取当前线程节点Node。检测是否中断，如果中断return null，等待后续抛出InterruptedException异常。
- 如果slot不为null，则进行slot消除，成功直接返回数据V，否则失败，则创建arena消除数组。
- 如果slot为null，但arena不为null，则返回null，进入arenaExchange逻辑。
- 如果slot为null，且arena也为null，则尝试占领该slot，失败重试，成功则跳出循环进入spin+block（自旋+阻塞）模式。

在自旋+阻塞模式中，首先取得结束时间和自旋次数。如果match(做releasing操作的线程传递的项)为null，其首先尝试spins+随机次自旋（改自旋使用当前节点中的hash，并改变之）和退让。

当自旋数为0后，假如slot发生了改变（slot != p）则重置自旋数并重试。否则假如：当前未中断&arena为null&（当前不是限时版本或者限时版本+当前时间未结束）：阻塞或者限时阻塞。假如：当前中断或者arena不为null或者当前为限时版本+时间已经结束：不限时版本：置v为null；限时版本：如果时间结束以及未中断则TIMED_OUT；否则给出null（原因是探测到arena非空或者当前线程中断）。 match不为空时跳出循环。 整个slotExchange清晰明了。 

**arenaExchange(Object item, boolean timed, long ns)**

```java
private final Object arenaExchange(Object item, boolean timed, long ns) {
    Node[] a = arena;
    Node p = participant.get();
    for (int i = p.index;;) {                      // access slot at i
        int b, m, c; long j;                       // j is raw array offset
        Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
        if (q != null && U.compareAndSwapObject(a, j, q, null)) {
            Object v = q.item;                     // release
            q.match = item;
            Thread w = q.parked;
            if (w != null)
                U.unpark(w);
            return v;
        }
        else if (i <= (m = (b = bound) & MMASK) && q == null) {
            p.item = item;                         // offer
            if (U.compareAndSwapObject(a, j, null, p)) {
                long end = (timed && m == 0) ? System.nanoTime() + ns : 0L;
                Thread t = Thread.currentThread(); // wait
                for (int h = p.hash, spins = SPINS;;) {
                    Object v = p.match;
                    if (v != null) {
                        U.putOrderedObject(p, MATCH, null);
                        p.item = null;             // clear for next use
                        p.hash = h;
                        return v;
                    }
                    else if (spins > 0) {
                        h ^= h << 1; h ^= h >>> 3; h ^= h << 10; // xorshift
                        if (h == 0)                // initialize hash
                            h = SPINS | (int)t.getId();
                        else if (h < 0 &&          // approx 50% true
                                 (--spins & ((SPINS >>> 1) - 1)) == 0)
                            Thread.yield();        // two yields per wait
                    }
                    else if (U.getObjectVolatile(a, j) != p)
                        spins = SPINS;       // releaser hasn't set match yet
                    else if (!t.isInterrupted() && m == 0 &&
                             (!timed ||
                              (ns = end - System.nanoTime()) > 0L)) {
                        U.putObject(t, BLOCKER, this); // emulate LockSupport
                        p.parked = t;              // minimize window
                        if (U.getObjectVolatile(a, j) == p)
                            U.park(false, ns);
                        p.parked = null;
                        U.putObject(t, BLOCKER, null);
                    }
                    else if (U.getObjectVolatile(a, j) == p &&
                             U.compareAndSwapObject(a, j, p, null)) {
                        if (m != 0)                // try to shrink
                            U.compareAndSwapInt(this, BOUND, b, b + SEQ - 1);
                        p.item = null;
                        p.hash = h;
                        i = p.index >>>= 1;        // descend
                        if (Thread.interrupted())
                            return null;
                        if (timed && m == 0 && ns <= 0L)
                            return TIMED_OUT;
                        break;                     // expired; restart
                    }
                }
            }
            else
                p.item = null;                     // clear offer
        }
        else {
            if (p.bound != b) {                    // stale; reset
                p.bound = b;
                p.collides = 0;
                i = (i != m || m == 0) ? m : m - 1;
            }
            else if ((c = p.collides) < m || m == FULL ||
                     !U.compareAndSwapInt(this, BOUND, b, b + SEQ + 1)) {
                p.collides = c + 1;
                i = (i == 0) ? m : i - 1;          // cyclically traverse
            }
            else
                i = m + 1;                         // grow
            p.index = i;
        }
    }
}
```

首先通过participant取得当前节点Node，然后根据当前节点Node的index去取arena中相对应的节点node。前面提到过arena可以确保不同的slot在arena中是不会相冲突的，那么是怎么保证的呢？我们先看arena的创建：

```java
arena = new Node[(FULL + 2) << ASHIFT];
```

这个arena到底有多大呢？我们先看FULL 和ASHIFT的定义：

```java
static final int FULL = (NCPU >= (MMASK << 1)) ? MMASK : NCPU >>> 1;
private static final int ASHIFT = 7;

private static final int NCPU = Runtime.getRuntime().availableProcessors();
private static final int MMASK = 0xff;      // 255
```

假如我的机器NCPU = 8 ，则得到的是768大小的arena数组。然后通过以下代码取得在arena中的节点：

```java
Node q = (Node)U.getObjectVolatile(a, j = (i << ASHIFT) + ABASE);
```

他仍然是通过右移ASHIFT位来取得Node的，ABASE定义如下：

```java
Class<?> ak = Node[].class;
ABASE = U.arrayBaseOffset(ak) + (1 << ASHIFT);
```

U.arrayBaseOffset获取对象头长度，数组元素的大小可以通过unsafe.arrayIndexScale(T[].class) 方法获取到。这也就是说要访问类型为T的第N个元素的话，你的偏移量offset应该是arrayOffset+N*arrayScale。也就是说BASE = arrayOffset+ 128 。其次我们再看Node节点的定义

```java
@sun.misc.Contended static final class Node{
  ....
}
```

### 小案例

```java
public class ExchangerTest {

    static class Producer implements Runnable{
        //生产者、消费者交换的数据结构
        private List<String> data;
        //步生产者和消费者的交换对象
        private Exchanger<List<String>> exchanger;
        Producer(List<String> data,Exchanger<List<String>> exchanger){
            this.data = data;
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            for(int i = 1 ; i < 5 ; i++){
                System.out.println("生产者第" + i + "次提供");
                for(int j = 1 ; j <= 3 ; j++){
                    System.out.println("生产者装入" + i  + "--" + j);
                    data.add("buffer：" + i + "--" + j);
                }

                System.out.println("生产者装满，等待与消费者交换...");
                try {
                    exchanger.exchange(data);
                } catch (Exception e) {
                    e.printStackTrace();
                }
            }
        }
    }

    static class Consumer implements Runnable {
        private List<String> data;
        private final Exchanger<List<String>> exchanger;
        public Consumer(List<String> data, Exchanger<List<String>> exchanger) {
            this.data = data;
            this.exchanger = exchanger;
        }

        @Override
        public void run() {
            for (int i = 1; i < 5; i++) {
                //调用exchange()与消费者进行数据交换
                try {
                    data = exchanger.exchange(data);
                } catch (InterruptedException e) {
                    e.printStackTrace();
                }

                System.out.println("消费者第" + i + "次提取");
                for (int j = 1; j <= 3 ; j++) {
                    System.out.println("消费者 : " + data.get(0));
                    data.remove(0);
                }
            }
        }
    }

    public static void main(String[] args) throws InterruptedException {
        List<String> buffer1 = new ArrayList<>();
        List<String> buffer2 = new ArrayList<>();

        Exchanger<List<String>> exchanger = new Exchanger<>();

        Thread producerThread = new Thread(new Producer(buffer1,exchanger));
        Thread consumerThread = new Thread(new Consumer(buffer2,exchanger));

        producerThread.start();
        consumerThread.start();
    }
}
```

首先生产者Producer、消费者Consumer首先都创建一个缓冲列表，通过Exchanger来同步交换数据。消费中通过调用Exchanger与生产者进行同步来获取数据，而生产者则通过for循环向缓存队列存储数据并使用exchanger对象消费者同步。到消费者从exchanger哪里得到数据后，他的缓冲列表中有3个数据，而生产者得到的则是一个空的列表。上面的例子充分展示了消费者-生产者是如何利用Exchanger来完成数据交换的。 在Exchanger中，如果一个线程已经到达了exchanger节点时，对于它的伙伴节点的情况有三种：

1. 如果它的伙伴节点在该线程到达之前已经调用了exchanger方法，则它会唤醒它的伙伴然后进行数据交换，得到各自数据返回。
2. 如果它的伙伴节点还没有到达交换点，则该线程将会被挂起，等待它的伙伴节点到达被唤醒，完成数据交换。
3. 如果当前线程被中断了则抛出异常，或者等待超时了，则抛出超时异常。

![](http://image.tinx.top/20211129145730.png)

# 并发非阻塞队列

> 实现的基础是CAS来实现的非阻塞

## ConcurrentHashMap

> HashMap是我们用得非常频繁的一个集合，但是由于它是非线程安全的，在多线程环境下，put操作是有可能产生死循环的，导致CPU利用率接近100%。为了解决该问题，提供了Hashtable和Collections.synchronizedMap(hashMap)两种解决方案，但是这两种方案都是对读写加锁，独占式，一个线程在读时其他线程必须等待，吞吐量较低，性能较为低下。故而Doug Lea大神给我们提供了高性能的线程安全HashMap：ConcurrentHashMap。

### 重要的常量

```java
// 最大容量：2^30=1073741824
private static final int MAXIMUM_CAPACITY = 1 << 30;
// 默认初始值，必须是2的幕数
private static final int DEFAULT_CAPACITY = 16;
// 最大的数组长度
static final int MAX_ARRAY_SIZE = Integer.MAX_VALUE - 8;
// 默认的并发等级 16个
private static final int DEFAULT_CONCURRENCY_LEVEL = 16;
// 加载因子 0.75
private static final float LOAD_FACTOR = 0.75f;
// 链表转红黑树阀值,> 8 链表转换为红黑树
static final int TREEIFY_THRESHOLD = 8;
//树转链表阀值，小于等于6（tranfer时，lc、hc=0两个计数器分别++记录原bin、新binTreeNode数量，<=UNTREEIFY_THRESHOLD 则untreeify(lo)）
static final int UNTREEIFY_THRESHOLD = 6;
// 最小的红黑树化的阈值
static final int MIN_TREEIFY_CAPACITY = 64;
// 最小转换阈值
private static final int MIN_TRANSFER_STRIDE = 16;
//
private static int RESIZE_STAMP_BITS = 16;
// 2^15-1，help resize的最大线程数
private static final int MAX_RESIZERS = (1 << (32 - RESIZE_STAMP_BITS)) - 1;
// 32-16=16，sizeCtl中记录size大小的偏移量
private static final int RESIZE_STAMP_SHIFT = 32 - RESIZE_STAMP_BITS;
// forwarding nodes的hash值
static final int MOVED     = -1;
// 树根节点的hash值
static final int TREEBIN   = -2;
// ReservationNode的hash值
static final int RESERVED  = -3;
// 可用处理器数量
static final int NCPU = Runtime.getRuntime().availableProcessors();
```

上面是ConcurrentHashMap定义的常量，简单易懂，就不多阐述了。下面介绍ConcurrentHashMap几个很重要的概念。

1. **table**：用来存放Node节点数据的，默认为null，默认大小为16的数组，每次扩容时大小总是2的幂次方；
2. **nextTable**：扩容时新生成的数据，数组为table的两倍；
3. **Node**：节点，保存key-value的数据结构；
4. **ForwardingNode**：一个特殊的Node节点，hash值为-1，其中存储nextTable的引用。只有table发生扩容的时候，ForwardingNode才会发挥作用，作为一个占位符放在table中表示当前节点为null或则已经被移动
5. **sizeCtl**：控制标识符，用来控制table初始化和扩容操作的，在不同的地方有不同的用途，其值也不同，所代表的含义也不同
   - 负数代表正在进行初始化或扩容操作
   - -1代表正在初始化
   - -N 表示有N-1个线程正在进行扩容操作
   - 正数或0代表hash表还没有被初始化，这个数值表示初始化或下一次进行扩容的大小

### 重要内部类

#### Node

作为ConcurrentHashMap中最核心、最重要的内部类，Node担负着重要角色：key-value键值对。所有插入ConCurrentHashMap的中数据都将会包装在Node中。定义如下：

```java
static class Node<K,V> implements Map.Entry<K,V> {
    final int hash;
    final K key;
    volatile V val;             //带有volatile，保证可见性
    volatile Node<K,V> next;    //下一个节点的指针

    Node(int hash, K key, V val, Node<K,V> next) {
        this.hash = hash;
        this.key = key;
        this.val = val;
        this.next = next;
    }

    public final K getKey()       { return key; }
    public final V getValue()     { return val; }
    public final int hashCode()   { return key.hashCode() ^ val.hashCode(); }
    public final String toString(){ return key + "=" + val; }
    /** 不允许修改value的值 */
    public final V setValue(V value) {
        throw new UnsupportedOperationException();
    }

    public final boolean equals(Object o) {
        Object k, v, u; Map.Entry<?,?> e;
        return ((o instanceof Map.Entry) &&
                (k = (e = (Map.Entry<?,?>)o).getKey()) != null &&
                (v = e.getValue()) != null &&
                (k == key || k.equals(key)) &&
                (v == (u = val) || v.equals(u)));
    }

    /**  赋值get()方法 */
    Node<K,V> find(int h, Object k) {
        Node<K,V> e = this;
        if (k != null) {
            do {
                K ek;
                if (e.hash == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
            } while ((e = e.next) != null);
        }
        return null;
    }
}
```

在Node内部类中，其属性value、next都是带有volatile的。同时其对value的setter方法进行了特殊处理，不允许直接调用其setter方法来修改value的值。最后Node还提供了find方法来赋值map.get()。

#### TreeNode

我们在学习HashMap的时候就知道，HashMap的核心数据结构就是链表。在ConcurrentHashMap中就不一样了，如果链表的数据过长是会转换为红黑树来处理。当它并不是直接转换，而是将这些链表的节点包装成TreeNode放在TreeBin对象中，然后由TreeBin完成红黑树的转换。所以TreeNode也必须是ConcurrentHashMap的一个核心类，其为树节点类，定义如下：

```java
static final class TreeNode<K,V> extends Node<K,V> {
    TreeNode<K,V> parent;  // red-black tree links
    TreeNode<K,V> left;
    TreeNode<K,V> right;
    TreeNode<K,V> prev;    // needed to unlink next upon deletion
    boolean red;

    TreeNode(int hash, K key, V val, Node<K,V> next,
             TreeNode<K,V> parent) {
        super(hash, key, val, next);
        this.parent = parent;
    }


    Node<K,V> find(int h, Object k) {
        return findTreeNode(h, k, null);
    }

    //查找hash为h，key为k的节点
    final TreeNode<K,V> findTreeNode(int h, Object k, Class<?> kc) {
        if (k != null) {
            TreeNode<K,V> p = this;
            do  {
                int ph, dir; K pk; TreeNode<K,V> q;
                TreeNode<K,V> pl = p.left, pr = p.right;
                if ((ph = p.hash) > h)
                    p = pl;
                else if (ph < h)
                    p = pr;
                else if ((pk = p.key) == k || (pk != null && k.equals(pk)))
                    return p;
                else if (pl == null)
                    p = pr;
                else if (pr == null)
                    p = pl;
                else if ((kc != null ||
                        (kc = comparableClassFor(k)) != null) &&
                        (dir = compareComparables(kc, k, pk)) != 0)
                    p = (dir < 0) ? pl : pr;
                else if ((q = pr.findTreeNode(h, k, kc)) != null)
                    return q;
                else
                    p = pl;
            } while (p != null);
        }
        return null;
    }
}
```

源码展示TreeNode继承Node，且提供了findTreeNode用来查找查找hash为h，key为k的节点。

#### TreeBin

该类并不负责key-value的键值对包装，它用于在链表转换为红黑树时包装TreeNode节点，也就是说ConcurrentHashMap红黑树存放是TreeBin，不是TreeNode。该类封装了一系列的方法，包括putTreeVal、lookRoot、UNlookRoot、remove、balanceInsetion、balanceDeletion。由于TreeBin的代码太长我们这里只展示构造方法（构造方法就是构造红黑树的过程）：

```java
static final class TreeBin<K,V> extends Node<K,V> {
    TreeNode<K, V> root;
    volatile TreeNode<K, V> first;
    volatile Thread waiter;
    volatile int lockState;
    static final int WRITER = 1; // set while holding write lock
    static final int WAITER = 2; // set when waiting for write lock
    static final int READER = 4; // increment value for setting read lock

    TreeBin(TreeNode<K, V> b) {
        super(TREEBIN, null, null, null);
        this.first = b;
        TreeNode<K, V> r = null;
        for (TreeNode<K, V> x = b, next; x != null; x = next) {
            next = (TreeNode<K, V>) x.next;
            x.left = x.right = null;
            if (r == null) {
                x.parent = null;
                x.red = false;
                r = x;
            } else {
                K k = x.key;
                int h = x.hash;
                Class<?> kc = null;
                for (TreeNode<K, V> p = r; ; ) {
                    int dir, ph;
                    K pk = p.key;
                    if ((ph = p.hash) > h)
                        dir = -1;
                    else if (ph < h)
                        dir = 1;
                    else if ((kc == null &&
                            (kc = comparableClassFor(k)) == null) ||
                            (dir = compareComparables(kc, k, pk)) == 0)
                        dir = tieBreakOrder(k, pk);
                    TreeNode<K, V> xp = p;
                    if ((p = (dir <= 0) ? p.left : p.right) == null) {
                        x.parent = xp;
                        if (dir <= 0)
                            xp.left = x;
                        else
                            xp.right = x;
                        r = balanceInsertion(r, x);
                        break;
                    }
                }
            }
        }
        this.root = r;
        assert checkInvariants(root);
    }

    /** 省略很多代码 */
}
```

通过构造方法是不是发现了部分端倪，构造方法就是在构造一个红黑树的过程。

#### ForwardingNode

这是一个真正的辅助类，该类仅仅只存活在ConcurrentHashMap扩容操作时。只是一个标志节点，并且指向nextTable，它提供find方法而已。该类也是集成Node节点，其hash为-1，key、value、next均为null。如下：

```java
static final class ForwardingNode<K,V> extends Node<K,V> {
    final Node<K,V>[] nextTable;
    ForwardingNode(Node<K,V>[] tab) {
        super(MOVED, null, null, null);
        this.nextTable = tab;
    }

    Node<K,V> find(int h, Object k) {
        // loop to avoid arbitrarily deep recursion on forwarding nodes
        outer: for (Node<K,V>[] tab = nextTable;;) {
            Node<K,V> e; int n;
            if (k == null || tab == null || (n = tab.length) == 0 ||
                    (e = tabAt(tab, (n - 1) & h)) == null)
                return null;
            for (;;) {
                int eh; K ek;
                if ((eh = e.hash) == h &&
                        ((ek = e.key) == k || (ek != null && k.equals(ek))))
                    return e;
                if (eh < 0) {
                    if (e instanceof ForwardingNode) {
                        tab = ((ForwardingNode<K,V>)e).nextTable;
                        continue outer;
                    }
                    else
                        return e.find(h, k);
                }
                if ((e = e.next) == null)
                    return null;
            }
        }
    }
}
```

### 构造函数

```java
public ConcurrentHashMap() {
}

public ConcurrentHashMap(int initialCapacity) {
    if (initialCapacity < 0) throw new IllegalArgumentException();
    int cap = ((initialCapacity >= (MAXIMUM_CAPACITY >>> 1)) ?
               MAXIMUM_CAPACITY :
               tableSizeFor(initialCapacity + (initialCapacity >>> 1) + 1));
    this.sizeCtl = cap;
}

public ConcurrentHashMap(Map<? extends K, ? extends V> m) {
    this.sizeCtl = DEFAULT_CAPACITY;
    putAll(m);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor) {
    this(initialCapacity, loadFactor, 1);
}

public ConcurrentHashMap(int initialCapacity, float loadFactor, int concurrencyLevel) {
    if (!(loadFactor > 0.0f) || initialCapacity < 0 || concurrencyLevel <= 0)
        throw new IllegalArgumentException();
    if (initialCapacity < concurrencyLevel) initialCapacity = concurrencyLevel;
    long size = (long)(1.0 + (long)initialCapacity / loadFactor);
    int cap = (size >= (long)MAXIMUM_CAPACITY) ?
        MAXIMUM_CAPACITY : tableSizeFor((int)size);
    this.sizeCtl = cap;
}
```

### 初始化 initTable()

ConcurrentHashMap的初始化主要由initTable()方法实现，在上面的构造函数中我们可以看到，其实ConcurrentHashMap在构造函数中并没有做什么事，仅仅只是设置了一些参数而已。其真正的初始化是发生在插入的时候，例如put、merge、compute、computeIfAbsent、computeIfPresent操作时。其方法定义如下：

```java
private final Node<K,V>[] initTable() {
    Node<K,V>[] tab; int sc;
    while ((tab = table) == null || tab.length == 0) {
        //sizeCtl < 0 表示有其他线程在初始化，该线程必须挂起
        if ((sc = sizeCtl) < 0) Thread.yield();
        // 如果该线程获取了初始化的权利，则用CAS将sizeCtl设置为-1，表示本线程正在初始化
        else if (U.compareAndSwapInt(this, SIZECTL, sc, -1)) {
            try {
                if ((tab = table) == null || tab.length == 0) {
                    int n = (sc > 0) ? sc : DEFAULT_CAPACITY;
                    Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n];
                    table = tab = nt;
                    // 下次扩容的大小
                    sc = n - (n >>> 2); ///相当于0.75*n 设置一个扩容的阈值  
                }
            } finally {
                sizeCtl = sc;
            }
            break;
        }
    }
    return tab;
}
```

初始化方法initTable()的关键就在于sizeCtl，该值默认为0，如果在构造函数时有参数传入该值则为2的幂次方。

该值如果 < 0，表示有其他线程正在初始化，则必须暂停该线程。如果线程获得了初始化的权限则先将sizeCtl设置为-1，防止有其他线程进入，最后将sizeCtl设置0.75 * n，表示扩容的阈值。

### Put操作

ConcurrentHashMap最常用的put、get操作，ConcurrentHashMap的put操作与HashMap并没有多大区别，其核心思想依然是根据hash值计算节点插入在table的位置，如果该位置为空，则直接插入，否则插入到链表或者树中。但是ConcurrentHashMap会涉及到多线程情况就会复杂很多。我们先看源代码，然后根据源代码一步一步分析：

```java
public V put(K key, V value) {
    return putVal(key, value, false);
}

final V putVal(K key, V value, boolean onlyIfAbsent) {
    //key、value均不能为null
    if (key == null || value == null) throw new NullPointerException();
    //计算hash值
    int hash = spread(key.hashCode());
    int binCount = 0;
    for (Node<K,V>[] tab = table;;) {
        Node<K,V> f; int n, i, fh;
        // table为null，进行初始化工作
        if (tab == null || (n = tab.length) == 0)
            tab = initTable();
        //如果i位置没有节点，则直接插入，不需要加锁
        else if ((f = tabAt(tab, i = (n - 1) & hash)) == null) {
            if (casTabAt(tab, i, null, new Node<K,V>(hash, key, value, null)))
                break;                   // no lock when adding to empty bin
        }
        // 有线程正在进行扩容操作，则先帮助扩容
        else if ((fh = f.hash) == MOVED)
            tab = helpTransfer(tab, f);
        else {
            V oldVal = null;
            //对该节点进行加锁处理（hash值相同的链表的头节点），对性能有点儿影响
            synchronized (f) {
                if (tabAt(tab, i) == f) {
                    //fh > 0 表示为链表，将该节点插入到链表尾部
                    if (fh >= 0) {
                        binCount = 1;
                        for (Node<K,V> e = f;; ++binCount) {
                            K ek;
                            //hash 和 key 都一样，替换value
                            if (e.hash == hash &&
                                    ((ek = e.key) == key ||
                                            (ek != null && key.equals(ek)))) {
                                oldVal = e.val;
                                //putIfAbsent()
                                if (!onlyIfAbsent)
                                    e.val = value;
                                break;
                            }
                            Node<K,V> pred = e;
                            //链表尾部  直接插入
                            if ((e = e.next) == null) {
                                pred.next = new Node<K,V>(hash, key,
                                        value, null);
                                break;
                            }
                        }
                    }
                    //树节点，按照树的插入操作进行插入
                    else if (f instanceof TreeBin) {
                        Node<K,V> p;
                        binCount = 2;
                        if ((p = ((TreeBin<K,V>)f).putTreeVal(hash, key,
                                value)) != null) {
                            oldVal = p.val;
                            if (!onlyIfAbsent)
                                p.val = value;
                        }
                    }
                }
            }
            if (binCount != 0) {
                // 如果链表长度已经达到临界值8 就需要把链表转换为树结构
                if (binCount >= TREEIFY_THRESHOLD)
                    treeifyBin(tab, i);
                if (oldVal != null)
                    return oldVal;
                break;
            }
        }
    }

    //size + 1  
    addCount(1L, binCount);
    return null;
}
```

按照上面的源码，我们可以确定put整个流程如下：

- 判空；ConcurrentHashMap的key、value都不允许为null
- 计算hash。利用方法计算hash值。

```java
static final int spread(int h) {
    return (h ^ (h >>> 16)) & HASH_BITS;
}
```

- 遍历table，进行节点插入操作，过程如下：
  - 如果table为空，则表示ConcurrentHashMap还没有初始化，则进行初始化操作：initTable()
  - 根据hash值获取节点的位置i，若该位置为空，则直接插入，这个过程是不需要加锁的。计算f位置：i=(n - 1) & hash
  - 如果检测到fh = f.hash == -1，则f是ForwardingNode节点，表示有其他线程正在进行扩容操作，则帮助线程一起进行扩容操作
  - 如果f.hash >= 0 表示是链表结构，则遍历链表，如果存在当前key节点则替换value，否则插入到链表尾部。如果f是TreeBin类型节点，则按照红黑树的方法更新或者增加节点
  - 若链表长度 > TREEIFY_THRESHOLD(默认是8)，则将链表转换为红黑树结构
- 调用addCount方法，ConcurrentHashMap的size + 1

### Get操作

ConcurrentHashMap的get操作还是挺简单的，无非就是通过hash来找key相同的节点而已，当然需要区分链表和树形两种情况。

```java
public V get(Object key) {
    Node<K,V>[] tab; Node<K,V> e, p; int n, eh; K ek;
    // 计算hash
    int h = spread(key.hashCode());
    if ((tab = table) != null && (n = tab.length) > 0 &&
            (e = tabAt(tab, (n - 1) & h)) != null) {
        // 搜索到的节点key与传入的key相同且不为null,直接返回这个节点
        if ((eh = e.hash) == h) {
            if ((ek = e.key) == key || (ek != null && key.equals(ek)))
                return e.val;
        }
        // 树
        else if (eh < 0) return (p = e.find(h, key)) != null ? p.val : null;
        // 链表，遍历
        while ((e = e.next) != null) {
            if (e.hash == h && ((ek = e.key) == key || (ek != null && key.equals(ek))))
                return e.val;
        }
    }
    return null;
}
```

get操作的整个逻辑非常清楚：

- 计算hash值
- 判断table是否为空，如果为空，直接返回null
- 根据hash值获取table中的Node节点（tabAt(tab, (n - 1) & h)），然后根据链表或者树形方式找到相对应的节点，返回其value值。

### Size操作

ConcurrentHashMap的size()方法我们虽然用得不是很多，但是我们还是很有必要去了解的。ConcurrentHashMap的size()方法返回的是一个不精确的值，因为在进行统计的时候有其他线程正在进行插入和删除操作。当然为了这个不精确的值，ConcurrentHashMap也是操碎了心。 为了更好地统计size，ConcurrentHashMap提供了baseCount、counterCells两个辅助变量和一个CounterCell辅助内部类。

```java
@sun.misc.Contended static final class CounterCell {
    volatile long value;
    CounterCell(long x) { value = x; }
}

//ConcurrentHashMap中元素个数,但返回的不一定是当前Map的真实元素个数。基于CAS无锁更新
private transient volatile long baseCount;

private transient volatile CounterCell[] counterCells;
```

这里我们需要清楚CounterCell 的定义 size()方法定义如下：

```java
public int size() {
    long n = sumCount();
    return ((n < 0L) ? 0 :
            (n > (long)Integer.MAX_VALUE) ? Integer.MAX_VALUE :
            (int)n);
}

final long sumCount() {
    CounterCell[] as = counterCells; CounterCell a;
    long sum = baseCount;
    if (as != null) {
        for (int i = 0; i < as.length; ++i) {
            //遍历，所有counter求和
            if ((a = as[i]) != null)
                sum += a.value;     
        }
    }
    return sum;
}
```

sumCount()就是迭代counterCells来统计sum的过程。我们知道put操作时，肯定会影响size()。 在put()方法最后会调用addCount()方法，该方法主要做两件事，一件更新baseCount的值，第二件检测是否进行扩容，我们只看更新baseCount部分：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // s = b + x，完成baseCount++操作；
    if ((as = counterCells) != null ||
        !U.compareAndSwapLong(this, BASECOUNT, b = baseCount, s = b + x)) {
        CounterCell a; long v; int m;
        boolean uncontended = true;
        if (as == null || (m = as.length - 1) < 0 ||
            (a = as[ThreadLocalRandom.getProbe() & m]) == null ||
            !(uncontended =
              U.compareAndSwapLong(a, CELLVALUE, v = a.value, v + x))) {
            //  多线程CAS发生失败时执行
            fullAddCount(x, uncontended);
            return;
        }
        if (check <= 1)
            return;
        s = sumCount();
    // 检查是否进行扩容
}
```

### 扩容操作

当ConcurrentHashMap中table元素个数达到了容量阈值（sizeCtl）时，则需要进行扩容操作。在put操作时最后一个会调用addCount(long x, int check)，该方法主要做两个工作：

1. 更新baseCount

2. 检测是否需要扩容操作。如下：

```java
private final void addCount(long x, int check) {
    CounterCell[] as; long b, s;
    // 更新baseCount

    //check >= 0 :则需要进行扩容操作
    if (check >= 0) {
        Node<K,V>[] tab, nt; int n, sc;
        while (s >= (long)(sc = sizeCtl) && (tab = table) != null &&
                (n = tab.length) < MAXIMUM_CAPACITY) {
            int rs = resizeStamp(n);
            if (sc < 0) {
                if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                        sc == rs + MAX_RESIZERS || (nt = nextTable) == null ||
                        transferIndex <= 0) break;
                if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1))
                    transfer(tab, nt);
            }

            //当前线程是唯一的或是第一个发起扩容的线程  此时nextTable=null
            else if (U.compareAndSwapInt(this, SIZECTL, sc, (rs << RESIZE_STAMP_SHIFT) + 2)) transfer(tab, null);
            s = sumCount();
        }
    }
}
```

transfer()方法为ConcurrentHashMap扩容操作的核心方法。由于ConcurrentHashMap支持多线程扩容，而且也没有进行加锁，所以实现会变得有点儿复杂。整个扩容操作分为两步：

1. 构建一个nextTable，其大小为原来大小的两倍，这个步骤是在单线程环境下完成的
2. 将原来table里面的内容复制到nextTable中，这个步骤是允许多线程操作的，所以性能得到提升，减少了扩容的时间消耗

我们先来看看源代码，然后再一步一步分析：

```java
private final void transfer(Node<K,V>[] tab, Node<K,V>[] nextTab) {
    int n = tab.length, stride;
    // 每核处理的量小于16，则强制赋值16
    if ((stride = (NCPU > 1) ? (n >>> 3) / NCPU : n) < MIN_TRANSFER_STRIDE)
        stride = MIN_TRANSFER_STRIDE; // subdivide range
    if (nextTab == null) {            // initiating
        try {
          	//构建一个nextTable对象，其容量为原来容量的两倍
            Node<K,V>[] nt = (Node<K,V>[])new Node<?,?>[n << 1];        
            nextTab = nt;
        } catch (Throwable ex) {      // try to cope with OOME
            sizeCtl = Integer.MAX_VALUE;
            return;
        }
        nextTable = nextTab;
        transferIndex = n;
    }
    int nextn = nextTab.length;
    // 连接点指针，用于标志位（fwd的hash值为-1，fwd.nextTable=nextTab）
    ForwardingNode<K,V> fwd = new ForwardingNode<K,V>(nextTab);
    // 当advance == true时，表明该节点已经处理过了
    boolean advance = true;
    boolean finishing = false; // to ensure sweep before committing nextTab
    for (int i = 0, bound = 0;;) {
        Node<K,V> f; int fh;
        // 控制 --i ,遍历原hash表中的节点
        while (advance) {
            int nextIndex, nextBound;
            if (--i >= bound || finishing)
                advance = false;
            else if ((nextIndex = transferIndex) <= 0) {
                i = -1;
                advance = false;
            }
            // 用CAS计算得到的transferIndex
            else if (U.compareAndSwapInt(this, TRANSFERINDEX, nextIndex,
                            nextBound = (nextIndex > stride ? nextIndex - stride : 0))) {
                bound = nextBound;
                i = nextIndex - 1;
                advance = false;
            }
        }
        if (i < 0 || i >= n || i + n >= nextn) {
            int sc;
            // 已经完成所有节点复制了
            if (finishing) {
                nextTable = null;
                table = nextTab;        // table 指向nextTable
                sizeCtl = (n << 1) - (n >>> 1);     // sizeCtl阈值为原来的1.5倍
                return;     // 跳出死循环，
            }
            // CAS 更扩容阈值，在这里面sizectl值减一，说明新加入一个线程参与到扩容操作
            if (U.compareAndSwapInt(this, SIZECTL, sc = sizeCtl, sc - 1)) {
                if ((sc - 2) != resizeStamp(n) << RESIZE_STAMP_SHIFT) return;
                finishing = advance = true;
                i = n; // recheck before commit
            }
        }
        // 遍历的节点为null，则放入到ForwardingNode 指针节点
        else if ((f = tabAt(tab, i)) == null) advance = casTabAt(tab, i, null, fwd);
        // f.hash == -1 表示遍历到了ForwardingNode节点，意味着该节点已经处理过了
        // 这里是控制并发扩容的核心
        else if ((fh = f.hash) == MOVED) advance = true; // already processed
        else {
            // 节点加锁
            synchronized (f) {
                // 节点复制工作
                if (tabAt(tab, i) == f) {
                    Node<K,V> ln, hn;
                    // fh >= 0 ,表示为链表节点
                    if (fh >= 0) {
                        // 构造两个链表  一个是原链表  另一个是原链表的反序排列
                        int runBit = fh & n;
                        Node<K,V> lastRun = f;
                        for (Node<K,V> p = f.next; p != null; p = p.next) {
                            int b = p.hash & n;
                            if (b != runBit) {
                                runBit = b;
                                lastRun = p;
                            }
                        }
                        if (runBit == 0) {
                            ln = lastRun;
                            hn = null;
                        }
                        else {
                            hn = lastRun;
                            ln = null;
                        }
                        for (Node<K,V> p = f; p != lastRun; p = p.next) {
                            int ph = p.hash; K pk = p.key; V pv = p.val;
                            if ((ph & n) == 0)
                                ln = new Node<K,V>(ph, pk, pv, ln);
                            else
                                hn = new Node<K,V>(ph, pk, pv, hn);
                        }
                        // 在nextTable i 位置处插上链表
                        setTabAt(nextTab, i, ln);
                        // 在nextTable i + n 位置处插上链表
                        setTabAt(nextTab, i + n, hn);
                        // 在table i 位置处插上ForwardingNode 表示该节点已经处理过了
                        setTabAt(tab, i, fwd);
                        // advance = true 可以执行--i动作，遍历节点
                        advance = true;
                    }
                    // 如果是TreeBin，则按照红黑树进行处理，处理逻辑与上面一致
                    else if (f instanceof TreeBin) {
                        TreeBin<K,V> t = (TreeBin<K,V>)f;
                        TreeNode<K,V> lo = null, loTail = null;
                        TreeNode<K,V> hi = null, hiTail = null;
                        int lc = 0, hc = 0;
                        for (Node<K,V> e = t.first; e != null; e = e.next) {
                            int h = e.hash;
                            TreeNode<K,V> p = new TreeNode<K,V>
                                    (h, e.key, e.val, null, null);
                            if ((h & n) == 0) {
                                if ((p.prev = loTail) == null)
                                    lo = p;
                                else
                                    loTail.next = p;
                                loTail = p;
                                ++lc;
                            }
                            else {
                                if ((p.prev = hiTail) == null)
                                    hi = p;
                                else
                                    hiTail.next = p;
                                hiTail = p;
                                ++hc;
                            }
                        }

                        // 扩容后树节点个数若<=6，将树转链表
                        ln = (lc <= UNTREEIFY_THRESHOLD) ? untreeify(lo) :
                                (hc != 0) ? new TreeBin<K,V>(lo) : t;
                        hn = (hc <= UNTREEIFY_THRESHOLD) ? untreeify(hi) :
                                (lc != 0) ? new TreeBin<K,V>(hi) : t;
                        setTabAt(nextTab, i, ln);
                        setTabAt(nextTab, i + n, hn);
                        setTabAt(tab, i, fwd);
                        advance = true;
                    }
                }
            }
        }
    }
}
```

上面的源码有点儿长，稍微复杂了一些，在这里我们抛弃它多线程环境，我们从单线程角度来看：

1. 为每个内核分任务，并保证其不小于16
2. 检查nextTable是否为null，如果是，则初始化nextTable，使其容量为table的两倍
3. 死循环遍历节点，知道finished：节点从table复制到nextTable中，支持并发，请思路如下：
   - 如果节点 f 为null，则插入ForwardingNode（采用Unsafe.compareAndSwapObjectf方法实现），这个是触发并发扩容的关键
   - 如果f为链表的头节点（fh >= 0）,则先构造一个反序链表，然后把他们分别放在nextTable的i和i + n位置，并将ForwardingNode 插入原节点位置，代表已经处理过了
   - 如果f为TreeBin节点，同样也是构造一个反序 ，同时需要判断是否需要进行unTreeify()操作，并把处理的结果分别插入到nextTable的i 和i+nw位置，并插入ForwardingNode 节点
4. 所有节点复制完成后，则将table指向nextTable，同时更新sizeCtl = nextTable的0.75倍，完成扩容过程

在多线程环境下，ConcurrentHashMap用两点来保证正确性：ForwardingNode和synchronized。当一个线程遍历到的节点如果是ForwardingNode，则继续往后遍历，如果不是，则将该节点加锁，防止其他线程进入，完成后设置ForwardingNode节点，以便要其他线程可以看到该节点已经处理过了，如此交叉进行，高效而又安全。 下图是扩容的过程

![](http://image.tinx.top/20211130131602.png)

在put操作时如果发现fh.hash = -1，则表示正在进行扩容操作，则当前线程会协助进行扩容操作。

```java
else if ((fh = f.hash) == MOVED) tab = helpTransfer(tab, f);
```

helpTransfer()方法为协助扩容方法，当调用该方法的时候，nextTable一定已经创建了，所以该方法主要则是进行复制工作。如下：

```java
final Node<K,V>[] helpTransfer(Node<K,V>[] tab, Node<K,V> f) {
    Node<K,V>[] nextTab; int sc;
    if (tab != null && (f instanceof ForwardingNode) &&
            (nextTab = ((ForwardingNode<K,V>)f).nextTable) != null) {
        int rs = resizeStamp(tab.length);
        while (nextTab == nextTable && table == tab &&
                (sc = sizeCtl) < 0) {
            if ((sc >>> RESIZE_STAMP_SHIFT) != rs || sc == rs + 1 ||
                    sc == rs + MAX_RESIZERS || transferIndex <= 0)
                break;
            if (U.compareAndSwapInt(this, SIZECTL, sc, sc + 1)) {
                transfer(tab, nextTab);
                break;
            }
        }
        return nextTab;
    }
    return table;
}
```

### 转换红黑树

在put操作是，如果发现链表结构中的元素超过了TREEIFY_THRESHOLD（默认为8），则会把链表转换为红黑树，已便于提高查询效率。如下：

```java
if (binCount >= TREEIFY_THRESHOLD)
  treeifyBin(tab, i);
```

调用treeifyBin方法用与将链表转换为红黑树。

```java
private final void treeifyBin(Node<K,V>[] tab, int index) {
    Node<K,V> b; int n, sc;
    if (tab != null) {
        if ((n = tab.length) < MIN_TREEIFY_CAPACITY)//如果table.length<64 就扩大一倍 返回
            tryPresize(n << 1);
        else if ((b = tabAt(tab, index)) != null && b.hash >= 0) {
            synchronized (b) {
                if (tabAt(tab, index) == b) {
                    TreeNode<K,V> hd = null, tl = null;
                    //构造了一个TreeBin对象 把所有Node节点包装成TreeNode放进去
                    for (Node<K,V> e = b; e != null; e = e.next) {
                      //这里只是利用了TreeNode封装 而没有利用TreeNode的next域和parent域
                        TreeNode<K,V> p = new TreeNode<K,V>(e.hash, e.key, e.val,null, null);
                        if ((p.prev = tl) == null) hd = p;
                        else tl.next = p;
                        tl = p;
                    }
                    //在原来index的位置 用TreeBin替换掉原来的Node对象
                    setTabAt(tab, index, new TreeBin<K,V>(hd));
                }
            }
        }
    }
}
```

从上面源码可以看出，构建红黑树的过程是同步的，进入同步后过程如下：

1. 根据table中index位置Node链表，重新生成一个hd为头结点的TreeNode
2. 根据hd头结点，生成TreeBin树结构，并用TreeBin替换掉原来的Node对象。

### 红黑树知识点

[详情请点击此](https://www.cmsblogs.com/article/1391297924817883136)

## ConcurrentLinkedQueue

> 要实现一个线程安全的队列有两种方式：阻塞和非阻塞。阻塞队列无非就是锁的应用，而非阻塞则是CAS算法的应用。下面我们就开始一个非阻塞算法的研究：CoucurrentLinkedQueue。 ConcurrentLinkedQueue是一个基于链接节点的无边界的线程安全队列，它采用FIFO原则对元素进行排序。采用“wait-free”算法（即CAS算法）来实现的。 CoucurrentLinkedQueue规定了如下几个不变性：
>
> 1. 在入队的最后一个元素的next为null
> 2. 队列中所有未删除的节点的item都不能为null且都能从head节点遍历到
> 3. 对于要删除的节点，不是直接将其设置为null，而是先将其item域设置为null（迭代器会跳过item为null的节点）
> 4. 允许head和tail更新滞后。这是什么意思呢？意思就说是head、tail不总是指向第一个元素和最后一个元素（后面阐述）。
>
> head的不变性和可变性：
>
> - 不变性
>   1. 所有未删除的节点都可以通过head节点遍历到
>   2. head不能为null
>   3. head节点的next不能指向自身
> - 可变性
>   1. head的item可能为null，也可能不为null
>   2. 允许tail滞后head，也就是说调用succc()方法，从head不可达tail
>
> tail的不变性和可变性
>
> - 不变性
>   1. tail不能为null
> - 可变性
>   1. tail的item可能为null，也可能不为null
>   2. tail节点的next域可以指向自身
>   3. 允许tail滞后head，也就是说调用succc()方法，从head不可达tail

### Node

CoucurrentLinkedQueue的结构由head节点和tail节点组成，每个节点由节点元素item和指向下一个节点的next引用组成，而节点与节点之间的关系就是通过该next关联起来的，从而组成一张链表的队列。节点Node为ConcurrentLinkedQueue的内部类，定义如下：

```java
private static class Node<E> {
    /** 节点元素域 */
    volatile E item;
    volatile Node<E> next;

    //初始化,获得item 和 next 的偏移量,为后期的CAS做准备

    Node(E item) {
        UNSAFE.putObject(this, itemOffset, item);
    }

    boolean casItem(E cmp, E val) {
        return UNSAFE.compareAndSwapObject(this, itemOffset, cmp, val);
    }

    void lazySetNext(Node<E> val) {
        UNSAFE.putOrderedObject(this, nextOffset, val);
    }

    boolean casNext(Node<E> cmp, Node<E> val) {
        return UNSAFE.compareAndSwapObject(this, nextOffset, cmp, val);
    }

    // Unsafe mechanics

    private static final sun.misc.Unsafe UNSAFE;
    /** 偏移量 */
    private static final long itemOffset;
    /** 下一个元素的偏移量 */

   private static final long nextOffset;

    static {
        try {
            UNSAFE = sun.misc.Unsafe.getUnsafe();
            Class<?> k = Node.class;
            itemOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("item"));
            nextOffset = UNSAFE.objectFieldOffset
                    (k.getDeclaredField("next"));
        } catch (Exception e) {
            throw new Error(e);
        }
    }
}
```

### 入列

入列，我们认为是一个非常简单的过程：

tail节点的next执行新节点，然后更新tail为新节点即可。

从单线程角度我们这么理解应该是没有问题的，但是多线程呢？如果一个线程正在进行插入动作，那么它必须先获取尾节点，然后设置尾节点的下一个节点为当前节点，但是如果已经有一个线程刚刚好完成了插入，那么尾节点是不是发生了变化？

对于这种情况ConcurrentLinkedQueue怎么处理呢？我们先看源码： offer(E e)：将指定元素插入都队列尾部：

```java
public boolean offer(E e) {
    //检查节点是否为null
    checkNotNull(e);
    // 创建新节点
    final Node<E> newNode = new Node<E>(e);

    //死循环 直到成功为止
    for (Node<E> t = tail, p = t;;) {
        Node<E> q = p.next;
        // q == null 表示 p已经是最后一个节点了，尝试加入到队列尾
        // 如果插入失败，则表示其他线程已经修改了p的指向
        if (q == null) {                                // --- 1
            // casNext：t节点的next指向当前节点
            // casTail：设置tail 尾节点
            if (p.casNext(null, newNode)) {             // --- 2
                // node 加入节点后会导致tail距离最后一个节点相差大于一个，需要更新tail
                if (p != t)                             // --- 3
                    casTail(t, newNode);                    // --- 4
                return true;
            }
        }
        // p == q 等于自身
        else if (p == q)                                // --- 5
            // p == q 代表着该节点已经被删除了
            // 由于多线程的原因，我们offer()的时候也会poll()，如果offer()的时候正好该节点已经poll()了
            // 那么在poll()方法中的updateHead()方法会将head指向当前的q，而把p.next指向自己，即：p.next == p
            // 这样就会导致tail节点滞后head（tail位于head的前面），则需要重新设置p
            p = (t != (t = tail)) ? t : head;           // --- 6
        // tail并没有指向尾节点
        else
            // tail已经不是最后一个节点，将p指向最后一个节点
            p = (p != t && t != (t = tail)) ? t : q;    // --- 7
    }
}
```

光看源码还是有点儿迷糊的，插入节点一次分析就会明朗很多。 

**初始化** ConcurrentLinkedQueue初始化时head、tail存储的元素都为null，且head等于tail：

<img src="http://image.tinx.top/image-20211130133150816.png" alt="image-20211130133150816" style="zoom: 50%;" />

**添加元素A**

按照程序分析：

第一次插入元素A，head = tail = dummyNode，所有q = p.next = null，直接走步骤2：p.casNext(null, newNode)，由于 p == t成立，所以不会执行步骤3：casTail(t, newNode)，直接return。插入A节点后如下：

<img src="http://image.tinx.top/image-20211130133235411.png" alt="image-20211130133235411" style="zoom:50%;" />



**添加元素B**

q = p.next = A ,p = tail = dummyNode，所以直接跳到步骤7：p = (p != t && t != (t = tail)) ? t : q;。此时p = q，然后进行第二次循环 q = p.next = null，步骤2：p == null成立，将该节点插入，因为p = q，t = tail，所以步骤3：p != t 成立，执行步骤4：casTail(t, newNode)，然后return。如下：

<img src="http://image.tinx.top/image-20211130133336819.png" alt="image-20211130133336819" style="zoom:50%;" />

**添加节点C**

此时t = tail ,p = t，q = p.next = null，和插入元素A无异，如下：

<img src="http://image.tinx.top/image-20211130133404151.png" alt="image-20211130133404151" style="zoom:50%;" />

这里整个offer()过程已经分析完成了，可能p == q 有点儿难理解，p 不是等于q.next么，怎么会有p == q呢？这个疑问我们在出列poll()中分析

### 出列

ConcurrentLinkedQueue提供了poll()方法进行出列操作。入列主要是涉及到tail，出列则涉及到head。我们先看源码：

```java
public E poll() {
    // 如果出现p被删除的情况需要从head重新开始
    restartFromHead:        // 这是什么语法？真心没有见过
    for (;;) {
        for (Node<E> h = head, p = h, q;;) {
            // 节点 item
            E item = p.item;
            // item 不为null，则将item 设置为null
            if (item != null && p.casItem(item, null)) {                    // --- 1
                // p != head 则更新head
                if (p != h)                                                 // --- 2
                    // p.next != null，则将head更新为p.next ,否则更新为p
                    updateHead(h, ((q = p.next) != null) ? q : p);          // --- 3
                return item;
            }
            // p.next == null 队列为空
            else if ((q = p.next) == null) {                                // --- 4
                updateHead(h, p);
                return null;
            }
            // 当一个线程在poll的时候，另一个线程已经把当前的p从队列中删除——将p.next = p，p已经被移除不能继续，需要重新开始
            else if (p == q)                                                // --- 5
                continue restartFromHead;
            else
                p = q;                                                      // --- 6
        }
    }
}
```

这个相对于offer()方法而言会简单些，里面有一个很重要的方法：updateHead()，该方法用于CAS更新head节点，如下：

```java
final void updateHead(Node<E> h, Node<E> p) {
    if (h != p && casHead(h, p))
        h.lazySetNext(h);
}
```

我们先将上面offer()的链表poll()掉，添加A、B、C节点结构如下：

<img src="http://image.tinx.top/image-20211130133705975.png" alt="image-20211130133705975" style="zoom:50%;" />

**poll A**

head = dumy，p = head， item = p.item = null，步骤1不成立，步骤4：(q = p.next) == null不成立，p.next = A，跳到步骤6，下一个循环，此时p = A，所以步骤1 item != null，进行p.casItem(item, null)成功，此时p == A != h，所以执行步骤3：updateHead(h, ((q = p.next) != null) ? q : p)，q = p.next = B != null，则将head CAS更新成B，如下：

<img src="http://image.tinx.top/image-20211130133735937.png" alt="image-20211130133735937" style="zoom:50%;" />

**poll B**

head = B ， p = head = B，item = p.item = B，步骤成立，步骤2：p != h 不成立，直接return，如下：

<img src="http://image.tinx.top/image-20211130133759826.png" alt="image-20211130133759826" style="zoom:50%;" />

**poll C**

head = dumy ，p = head = dumy，tiem = p.item = null，步骤1不成立，跳到步骤4：(q = p.next) == null，不成立，然后跳到步骤6，此时，p = q = C，item = C(item)，步骤1成立，所以讲C（item）设置为null，步骤2：p != h成立，执行步骤3：updateHead(h, ((q = p.next) != null) ? q : p)，如下：

<img src="http://image.tinx.top/image-20211130133826847.png" alt="image-20211130133826847" style="zoom:50%;" />

看到这里是不是一目了然了，在这里我们再来分析offer()的步骤5：

```java
else if(p == q){
    p = (t != (t = tail))? t : head;
}
```

ConcurrentLinkedQueue中规定，p == q表明，该节点已经被删除了，也就说tail滞后于head，head无法通过succ()方法遍历到tail，怎么做？ (t != (t = tail))? t : head;（这段代码的可读性实在是太差了，真他妈难理解：不知道是否可以理解为t != tail ? tail : head）这段代码主要是来判读tail节点是否已经发生了改变，如果发生了改变，则说明tail已经重新定位了，只需要重新找到tail即可，否则就只能指向head了。 就上面那个我们再次插入一个元素D。则p = head，q = p.next = null，执行步骤1： q = null且 p != t ，所以执行步骤4:，如下：

<img src="http://image.tinx.top/image-20211130133920637.png" alt="image-20211130133920637" style="zoom:50%;" />

再插入元素E，q = p.next = null，p == t，所以插入E后如下：

<img src="http://image.tinx.top/image-20211130133954823.png" alt="image-20211130133954823" style="zoom:50%;" />

## ConcurrentSkipListMap

### SkipList

什么是SkipList？Skip List ，称之为跳表，它是一种可以替代平衡树的数据结构，其数据元素默认按照key值升序，天然有序。Skip list让已排序的数据分布在多层链表中，以0-1随机数决定一个数据的向上攀升与否，通过“空间来换取时间”的一个算法，在每个节点中增加了向前的指针，在插入、删除、查找时可以忽略一些不可能涉及到的结点，从而提高了效率。 我们先看一个简单的链表，如下：

<img src="http://image.tinx.top/image-20211130134148125.png" alt="image-20211130134148125" style="zoom:50%;" />

如果我们需要查询9、21、30，则需要比较次数为3 + 6 + 8 = 17 次，那么有没有优化方案呢？有！我们将该链表中的某些元素提炼出来作为一个比较“索引”，如下：

<img src="http://image.tinx.top/image-20211130134211773.png" alt="image-20211130134211773" style="zoom:50%;" />

我们先与这些索引进行比较来决定下一个元素是往右还是下走，由于存在“索引”的缘故，导致在检索的时候会大大减少比较的次数。当然元素不是很多，很难体现出优势，当元素足够多的时候，这种索引结构就会大显身手。

### SkipList特性

SkipList具备如下特性：

1. 由很多层结构组成，level是通过一定的概率随机产生的
2. 每一层都是一个有序的链表，默认是升序，也可以根据创建映射时所提供的Comparator进行排序，具体取决于使用的构造方法
3. 最底层(Level 1)的链表包含所有元素
4. 如果一个元素出现在Level i 的链表中，则它在Level i 之下的链表也都会出现
5. 每个节点包含两个指针，一个指向同一链表中的下一个元素，一个指向下面一层的元素

我们将上图再做一些扩展就可以变成一个典型的SkipList结构了

<img src="http://image.tinx.top/image-20211130134456012.png" alt="image-20211130134456012" style="zoom:50%;" />

### SkipList的查找

SkipListd的查找算法较为简单，对于上面我们我们要查找元素21，其过程如下：

1. 比较3，大于，往后找（9），
2. 比9大，继续往后找（25），但是比25小，则从9的下一层开始找（16）
3. 16的后面节点依然为25，则继续从16的下一层找
4. 找到21

<img src="http://image.tinx.top/image-20211130134752474.png" alt="image-20211130134752474" style="zoom:50%;" />

### SkipList的插入

SkipList的插入操作主要包括：

1. 查找合适的位置。这里需要明确一点就是在确认新节点要占据的层次K时，采用丢硬币的方式，完全随机。如果占据的层次K大于链表的层次，则重新申请新的层，否则插入指定层次
2. 申请新的节点
3. 调整指针

假定我们要插入的元素为23，经过查找可以确认她是位于25后，9、16、21前。当然需要考虑申请的层次K。 **如果层次K > 3** 需要申请新层次（Level 4）

<img src="http://image.tinx.top/image-20211130134905095.png" alt="image-20211130134905095" style="zoom:50%;" />

**如果层次 K = 2** 直接在Level 2 层插入即可

<img src="http://image.tinx.top/image-20211130135031206.png" alt="image-20211130135031206" style="zoom:50%;" />

### SkipList的删除

删除节点和插入节点思路基本一致：找到节点，删除节点，调整指针。 比如删除节点9，如下：

<img src="http://image.tinx.top/image-20211130135510931.png" alt="image-20211130135510931" style="zoom:50%;" />

通过上面我们知道SkipList采用空间换时间的算法，其插入和查找的效率O(logn)，其效率不低于红黑树，但是其原理和实现的复杂度要比红黑树简单多了。一般来说会操作链表List，就会对SkipList毫无压力。 

ConcurrentSkipListMap其内部采用SkipLis数据结构实现。

为了实现SkipList，ConcurrentSkipListMap提供了三个内部类来构建这样的链表结构：

+ Node 表示最底层的单链表有序节点
+ Index 表示为基于Node的索引层
+ HeadIndex 用来维护索引层次

### Node

```java
static final class Node<K,V> {
    final K key;
    volatile Object value;
    volatile ConcurrentSkipListMap.Node<K, V> next;

    /** 省略些许代码 */
}
```

Node的结构和一般的单链表毫无区别，key-value和一个指向下一个节点的next。 

### Index

```java
static class Index<K,V> {
    final ConcurrentSkipListMap.Node<K,V> node;
    final ConcurrentSkipListMap.Index<K,V> down;
    volatile ConcurrentSkipListMap.Index<K,V> right;

    /** 省略些许代码 */
}
```

Index提供了一个基于Node节点的索引Node，一个指向下一个Index的right，一个指向下层的down节点。 

### **HeadIndex**

```java
static final class HeadIndex<K,V> extends Index<K,V> {
    final int level;  //索引层，从1开始，Node单链表层为0
    HeadIndex(Node<K,V> node, Index<K,V> down, Index<K,V> right, int level) {
        super(node, down, right);
        this.level = level;
    }
}
```

HeadIndex内部就一个level来定义层级。 ConcurrentSkipListMap提供了四个构造函数，每个构造函数都会调用initialize()方法进行初始化工作。

```java
final void initialize() {
    keySet = null;
    entrySet = null;
    values = null;
    descendingMap = null;
    randomSeed = seedGenerator.nextInt() | 0x0100; // ensure nonzero
    head = new ConcurrentSkipListMap.HeadIndex<K,V>(new ConcurrentSkipListMap.Node<K,V>(null, BASE_HEADER, null),
            null, null, 1);
}
```

注意，initialize()方法不仅仅只在构造函数中被调用，如clone，clear、readObject时都会调用该方法进行初始化步骤。这里需要注意randomSeed的初始化。

```java
private transient int randomSeed;
randomSeed = seedGenerator.nextInt() | 0x0100; // ensure nonzero
```

randomSeed一个简单的随机数生成器（在后面介绍）。

### Put操作

CoucurrentSkipListMap提供了put()方法用于将指定值与此映射中的指定键关联。源码如下：

```java
public V put(K key, V value) {
    if (value == null)
        throw new NullPointerException();
    return doPut(key, value, false);
}
```

首先判断value如果为null，则抛出NullPointerException，否则调用doPut方法，其实如果各位看过JDK的源码的话，应该对这样的操作很熟悉了，JDK源码里面很多方法都是先做一些必要性的验证后，然后通过调用do**()方法进行真正的操作。 doPut()方法内容较多，我们分步分析。

```java
private V doPut(K key, V value, boolean onlyIfAbsent) {
	Node<K,V> z;             // added node
	if (key == null)
	throw new NullPointerException();
	// 比较器
	Comparator<? super K> cmp = comparator;
	outer: for (;;) {
	for (Node<K, V> b = findPredecessor(key, cmp), n = b.next; ; ) {
	
	/** 省略代码 */
	}
}
```

doPut()方法有三个参数，除了key,value外还有一个boolean类型的onlyIfAbsent，该参数作用与如果存在当前key时，该做何动作。当onlyIfAbsent为false时，替换value，为true时，则返回该value。用代码解释为：

```java
if (!map.containsKey(key))
    return map.put(key, RbCvalue);
else
     return map.get(key);
```

首先判断key是否为null，如果为null，则抛出NullPointerException，从这里我们可以确认ConcurrentSkipList是不支持key或者value为null的。然后调用findPredecessor()方法，传入key来确认位置。findPredecessor()方法其实就是确认key要插入的位置。

```java
private Node<K,V> findPredecessor(Object key, Comparator<? super K> cmp) {
    if (key == null)
        throw new NullPointerException(); // don't postpone errors
    for (;;) {
        // 从head节点开始，head是level最高级别的headIndex
        for (Index<K,V> q = head, r = q.right, d;;) {

            // r != null，表示该节点右边还有节点，需要比较
            if (r != null) {
                Node<K,V> n = r.node;
                K k = n.key;
                // value == null，表示该节点已经被删除了
                // 通过unlink()方法过滤掉该节点
                if (n.value == null) {
                    //删掉r节点
                    if (!q.unlink(r))
                        break;           // restart
                    r = q.right;         // reread r
                    continue;
                }
                // value != null，节点存在
                // 如果key 大于r节点的key 则往前进一步
                if (cpr(cmp, key, k) > 0) {
                    q = r;
                    r = r.right;
                    continue;
                }
            }
            // 到达最右边，如果dowm == null，表示指针已经达到最下层了，直接返回该节点
            if ((d = q.down) == null) return q.node;
            q = d;
            r = d.right;
        }
    }
}
```

findPredecessor()方法意思非常明确：寻找前辈。从最高层的headIndex开始向右一步一步比较，直到right为null或者右边节点的Node的key大于当前key为止，然后再向下寻找，依次重复该过程，直到down为null为止，即找到了前辈，看返回的结果注意是Node，不是Item，所以插入的位置应该是最底层的Node链表。 在这个过程中ConcurrentSkipListMap赋予了该方法一个其他的功能，就是通过判断节点的value是否为null，如果为null，表示该节点已经被删除了，通过调用unlink()方法删除该节点。

```java
final boolean unlink(Index<K,V> succ) {
    return node.value != null && casRight(succ, succ.right);
}
```

删除节点过程非常简单，更改下right指针即可。 通过findPredecessor()找到前辈节点后，做什么呢？看下面：

```java
 for (Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
    // 前辈节点的next != null
    if (n != null) {
        Object v; int c;
        Node<K,V> f = n.next;

        // 不一致读，主要原因是并发，有节点捷足先登
        if (n != b.next)               // inconsistent read
            break;

        // n.value == null，该节点已经被删除了
        if ((v = n.value) == null) {   // n is deleted
            n.helpDelete(b, f);
            break;
        }

        // 前辈节点b已经被删除
        if (b.value == null || v == n) // b is deleted
            break;

        // 节点大于，往前移
        if ((c = cpr(cmp, key, n.key)) > 0) {
            b = n;
            n = f;
            continue;
        }

        // c == 0 表示，找到一个key相等的节点，根据onlyIfAbsent参数来做判断
        // onlyIfAbsent ==false，则通过casValue，替换value
        // onlyIfAbsent == true，返回该value
        if (c == 0) {
            if (onlyIfAbsent || n.casValue(v, value)) {
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
            break; // restart if lost race to replace value
        }
        // else c < 0; fall through
    }

    // 将key-value包装成一个node，插入
    z = new Node<K,V>(key, value, n);
    if (!b.casNext(n, z))
        break;         // restart if lost race to append to b
    break outer;
}
```

找到合适的位置后，就是在该位置插入节点咯。插入节点的过程比较简单，就是将key-value包装成一个Node，然后通过casNext()方法加入到链表当中。当然是插入之前需要进行一系列的校验工作。 在最下层插入节点后，下一步工作是什么？新建索引。前面博主提过，在插入节点的时候，会根据采用抛硬币的方式来决定新节点所插入的层次，由于存在并发的可能，ConcurrentSkipListMap采用ThreadLocalRandom来生成随机数。如下：

```java
int rnd = ThreadLocalRandom.nextSecondarySeed();
```

抛硬币决定层次的思想很简单，就是通过抛硬币如果硬币为正面则层次level + 1 ，否则停止，如下：

```java
// 抛硬币决定层次
while (((rnd >>>= 1) & 1) != 0)
    ++level;
```

在阐述SkipList插入节点的时候说明了，决定的层次level会分为两种情况进行处理，一是如果层次level大于最大的层次话则需要新增一层，否则就在相应层次以及小于该level的层次进行节点新增处理。

 **level <= headIndex.level**

```java
// 如果决定的层次level比最高层次head.level小，直接生成最高层次的index
// 由于需要确认每一层次的down，所以需要从最下层依次往上生成
if (level <= (max = h.level)) {
    for (int i = 1; i <= level; ++i)
        idx = new ConcurrentSkipListMap.Index<K,V>(z, idx, null);
}
```

从底层开始，小于level的每一层都初始化一个index，每次的node都指向新加入的node，down指向下一层的item，右侧next全部为null。整个处理过程非常简单：为小于level的每一层初始化一个index，然后加入到原来的index链条中去。 

**level > headIndex.level**

```java
// leve > head.level 则新增一层
else { // try to grow by one level
    // 新增一层
    level = max + 1;

    // 初始化 level个item节点
    @SuppressWarnings("unchecked")
    ConcurrentSkipListMap.Index<K,V>[] idxs =
            (ConcurrentSkipListMap.Index<K,V>[])new ConcurrentSkipListMap.Index<?,?>[level+1];
    for (int i = 1; i <= level; ++i)
        idxs[i] = idx = new ConcurrentSkipListMap.Index<K,V>(z, idx, null);

    //
    for (;;) {
        h = head;
        int oldLevel = h.level;
        // 层次扩大了，需要重新开始（有新线程节点加入）
        if (level <= oldLevel) // lost race to add level
            break;
        // 新的头结点HeadIndex
        ConcurrentSkipListMap.HeadIndex<K,V> newh = h;
        ConcurrentSkipListMap.Node<K,V> oldbase = h.node;
        // 生成新的HeadIndex节点，该HeadIndex指向新增层次
        for (int j = oldLevel+1; j <= level; ++j)
            newh = new ConcurrentSkipListMap.HeadIndex<K,V>(oldbase, newh, idxs[j], j);

        // HeadIndex CAS替换
        if (casHead(h, newh)) {
            h = newh;
            idx = idxs[level = oldLevel];
            break;
        }
    }
```

当抛硬币决定的level大于最大层次level时，需要新增一层进行处理。处理逻辑如下：

1. 初始化一个对应的index数组，大小为level + 1，然后为每个单位都创建一个index，个中参数为：Node为新增的Z，down为下一层index，right为null
2. 通过for循环来进行扩容操作。从最高层进行处理，新增一个HeadIndex，个中参数：节点Node，down都为最高层的Node和HeadIndex，right为刚刚创建的对应层次的index，level为相对应的层次level。最后通过CAS把当前的head与新加入层的head进行替换。

通过上面步骤我们发现，尽管已经找到了前辈节点，也将node插入了，也确定确定了层次并生成了相应的Index，但是并没有将这些Index插入到相应的层次当中，所以下面的代码就是将index插入到相对应的层当中。

```java
// 从插入的层次level开始
splice: for (int insertionLevel = level;;) {
    int j = h.level;
    //  从headIndex开始
    for (ConcurrentSkipListMap.Index<K,V> q = h, r = q.right, t = idx;;) {
        if (q == null || t == null)
            break splice;

        // r != null；这里是找到相应层次的插入节点位置，注意这里只横向找
        if (r != null) {
            ConcurrentSkipListMap.Node<K,V> n = r.node;

            int c = cpr(cmp, key, n.key);

            // n.value == null ，解除关系，r右移
            if (n.value == null) {
                if (!q.unlink(r))
                    break;
                r = q.right;
                continue;
            }

            // key > n.key 右移
            if (c > 0) {
                q = r;
                r = r.right;
                continue;
            }
        }

        // 上面找到节点要插入的位置，这里就插入
        // 当前层是最顶层
        if (j == insertionLevel) {
            // 建立联系
            if (!q.link(r, t))
                break; // restart
            if (t.node.value == null) {
                findNode(key);
                break splice;
            }
            // 标志的插入层 -- ，如果== 0 ，表示已经到底了，插入完毕，退出循环
            if (--insertionLevel == 0)
                break splice;
        }

        // 上面节点已经插入完毕了，插入下一个节点
        if (--j >= insertionLevel && j < level)
            t = t.down;
        q = q.down;
        r = q.right;
    }
}
```

这段代码分为两部分看，一部分是找到相应层次的该节点插入的位置，第二部分在该位置插入，然后下移。 至此，ConcurrentSkipListMap的put操作到此就结束了。代码量有点儿多，这里总结下：

1. 首先通过findPredecessor()方法找到前辈节点Node
2. 根据返回的前辈节点以及key-value，新建Node节点，同时通过CAS设置next
3. 设置节点Node，再设置索引节点。采取抛硬币方式决定层次，如果所决定的层次大于现存的最大层次，则新增一层，然后新建一个Item链表。
4. 最后，将新建的Item链表插入到SkipList结构中。
### Get操作

相比于put操作 ，get操作会简单很多，其过程其实就只相当于put操作的第一步：

```java
private V doGet(Object key) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        for (ConcurrentSkipListMap.Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null)
                break outer;
            ConcurrentSkipListMap.Node<K,V> f = n.next;
            if (n != b.next)                // inconsistent read
                break;
            if ((v = n.value) == null) {    // n is deleted
                n.helpDelete(b, f);
                break;
            }
            if (b.value == null || v == n)  // b is deleted
                break;
            if ((c = cpr(cmp, key, n.key)) == 0) {
                @SuppressWarnings("unchecked") V vv = (V)v;
                return vv;
            }
            if (c < 0)
                break outer;
            b = n;
            n = f;
        }
    }
    return null;
}
```

与put操作第一步相似，首先调用findPredecessor()方法找到前辈节点，然后顺着right一直往右找即可，同时在这个过程中同样承担了一个删除value为null的节点的职责。
### Remove操作

remove操作为删除指定key节点，如下：

```java
public V remove(Object key) {
    return doRemove(key, null);
}
```

直接调用doRemove()方法，这里remove有两个参数，一个是key，另外一个是value，所以doRemove方法即提供remove key，也提供同时满足key-value。

```java
final V doRemove(Object key, Object value) {
    if (key == null)
        throw new NullPointerException();
    Comparator<? super K> cmp = comparator;
    outer: for (;;) {
        for (ConcurrentSkipListMap.Node<K,V> b = findPredecessor(key, cmp), n = b.next;;) {
            Object v; int c;
            if (n == null)
                break outer;
            ConcurrentSkipListMap.Node<K,V> f = n.next;

            // 不一致读，重新开始
            if (n != b.next)                    // inconsistent read
                break;

            // n节点已删除
            if ((v = n.value) == null) {        // n is deleted
                n.helpDelete(b, f);
                break;
            }

            // b节点已删除
            if (b.value == null || v == n)      // b is deleted
                break;

            if ((c = cpr(cmp, key, n.key)) < 0)
                break outer;

            // 右移
            if (c > 0) {
                b = n;
                n = f;
                continue;
            }

            /*
             * 找到节点
             */

            // value != null 表示需要同时校验key-value值
            if (value != null && !value.equals(v))
                break outer;

            // CAS替换value
            if (!n.casValue(v, null))
                break;
            if (!n.appendMarker(f) || !b.casNext(n, f))
                findNode(key);                  // retry via findNode
            else {
                // 清理节点
                findPredecessor(key, cmp);      // clean index

                // head.right == null表示该层已经没有节点，删掉该层
                if (head.right == null)
                    tryReduceLevel();
            }
            @SuppressWarnings("unchecked") V vv = (V)v;
            return vv;
        }
    }
    return null;
}
```

调用findPredecessor()方法找到前辈节点，然后通过右移，然后比较，找到后利用CAS把value替换为null，然后判断该节点是不是这层唯一的index，如果是的话，调用tryReduceLevel()方法把这层干掉，完成删除。 其实从这里可以看出，remove方法仅仅是把Node的value设置null，并没有真正删除该节点Node，其实从上面的put操作、get操作我们可以看出，他们在寻找节点的时候都会判断节点的value是否为null，如果为null，则调用unLink()方法取消关联关系，如下：

```java
if (n.value == null) {
    if (!q.unlink(r)) break;           // restart
    r = q.right;         // reread r
    continue;
}
```

### Size操作

ConcurrentSkipListMap的size()操作和ConcurrentHashMap不同，它并没有维护一个全局变量来统计元素的个数，所以每次调用该方法的时候都需要去遍历。

```java
public int size() {
    long count = 0;
    for (Node<K,V> n = findFirst(); n != null; n = n.next) {
        if (n.getValidValue() != null)
            ++count;
    }
    return (count >= Integer.MAX_VALUE) ? Integer.MAX_VALUE : (int) count;
}
```

调用findFirst()方法找到第一个Node，然后利用node的next去统计。最后返回统计数据，最多能返回Integer.MAX_VALUE。注意这里在线程并发下是安全的。 ConcurrentSkipListMap过程其实不复杂，相比于ConcurrentHashMap而言，是简单的不能再简单了。对跳表SkipList熟悉的话，ConcurrentSkipListMap 应该是盘中餐了。 

# 并发阻塞队列

## BlockingQueue

BlockingQueue接口实现Queue接口，它支持两个附加操作：获取元素时等待队列变为非空，以及存储元素时等待空间变得可用。相对于同一操作他提供了四种机制：抛出异常、返回特殊值、阻塞等待、超时：

![image-20211130144503414](http://image.tinx.top/image-20211130144503414.png)

BlockingQueue常用于生产者和消费者场景。

JDK 8 中提供了七个阻塞队列可供使用（上图的DelayedWorkQueue是ScheduledThreadPoolExecutor的内部类）：

+ ArrayBlockingQueue ：一个由数组结构组成的有界阻塞队列。
+ LinkedBlockingQueue ：一个由链表结构组成的无界阻塞队列。
+ PriorityBlockingQueue ：一个支持优先级排序的无界阻塞队列。
+ DelayQueue：一个使用优先级队列实现的无界阻塞队列。
+ SynchronousQueue：一个不存储元素的阻塞队列。
+ LinkedTransferQueue：一个由链表结构组成的无界阻塞队列。
+ LinkedBlockingDeque：一个由链表结构组成的双向阻塞队列。

### ArrayBlockingQueue

基于数组的阻塞队列，ArrayBlockingQueue内部维护这一个定长数组，阻塞队列的大小在初始化时就已经确定了，其后无法更改。

采用可重入锁ReentrantLock来保证线程安全性，但是生产者和消费者是共用同一个锁对象，这样势必会导致降低一定的吞吐量。当然ArrayBlockingQueue完全可以采用分离锁来实现生产者和消费者的并行操作，但是我认为这样做只会给代码带来额外的复杂性，对于性能而言应该不会有太大的提升，因为基于数组的ArrayBlockingQueue在数据的写入和读取操作已经非常轻巧了。

ArrayBlockingQueue支持公平性和非公平性，默认采用非公平模式，可以通过构造函数设置为公平访问策略（true）。

### PriorityBlockingQueue

> PriorityBlockingQueue是一个支持优先级的无界阻塞队列。默认情况下元素采用自然顺序升序排序，当然我们也可以通过构造函数来指定Comparator来对元素进行排序。需要注意的是PriorityBlockingQueue不能保证同优先级元素的顺序。

PriorityBlockingQueue是支持优先级的无界队列。默认情况下采用自然顺序排序，当然也可以通过自定义Comparator来指定元素的排序顺序。

PriorityBlockingQueue内部采用二叉堆的实现方式，整个处理过程并不是特别复杂。添加操作则是不断“上冒”，而删除操作则是不断“下掉”。

#### 二叉堆

二叉堆是一种特殊的堆，就结构性而言就是完全二叉树或者是近似完全二叉树，满足树结构性和堆序性。树机构特性就是完全二叉树应该有的结构

堆序性则是：父节点的键值总是保持固定的序关系于任何一个子节点的键值，且每个节点的左子树和右子树都是一个二叉堆。

它有两种表现形式：最大堆、最小堆。 最大堆：父节点的键值总是大于或等于任何一个子节点的键值（下右图） 最小堆：父节点的键值总是小于或等于任何一个子节点的键值（下走图）

<img src="http://image.tinx.top/image-20211201105219566.png" alt="image-20211201105219566" style="zoom:50%;float:left;" />

二叉堆一般用数组表示，如果父节点的节点位置在n处，那么其左孩子节点为：2 * n + 1 ，其右孩子节点为2 * (n + 1)，其父节点为（n - 1） / 2 处。上左图的数组表现形式为：

<img src="http://image.tinx.top/image-20211201105350631.png" alt="image-20211201105350631" style="zoom:50%;float:left;" />

二叉堆的基本结构了解了，下面来看看二叉堆的添加和删除节点。二叉堆的添加和删除相对于二叉树来说会简单很多。

##### 添加元素

首先将要添加的元素N插添加到堆的末尾位置（在二叉堆中我们称之为空穴）。

如果元素N放入空穴中而不破坏堆的序（其值大于跟父节点值（最大堆是小于父节点）），那么插入完成。否则，我们则将该元素N的节点与其父节点进行交换，然后与其新父节点进行比较直到它的父节点不在比它小（最大堆是大）或者到达根节点。 假如有如下一个二叉堆

<img src="http://image.tinx.top/image-20211201105525706.png" alt="image-20211201105525706" style="zoom:50%;float:left;" />

这是一个最小堆，其父节点总是小于等于任一一个子节点。现在我们添加一个元素2。 第一步：在末尾添加一个元素2，如下：

<img src="http://image.tinx.top/image-20211201105622200.png" alt="image-20211201105622200" style="zoom:50%;float:left;" />

第二步：元素2比其父节点6小，进行替换，如下：

<img src="http://image.tinx.top/image-20211201105653871.png" alt="image-20211201105653871" style="zoom:50%;float:left;" />

第三步：继续与其父节点5比较，小于，替换：

<img src="http://image.tinx.top/image-20211201105727777.png" alt="image-20211201105727777" style="zoom:50%;float:left;" />

第四步：继续比较其跟节点1，发现跟节点比自己小，则完成，到这里元素2插入完毕。所以整个添加元素过程可以概括为：在元素末尾插入元素，然后不断比较替换直到不能移动为止。 复杂度：Ο(logn)

##### 删除元素

删除元素与增加元素一样，需要维护整个二叉堆的序。删除位置1的元素（数组下标0），则把最后一个元素空出来移到最前边，然后和它的两个子节点比较，如果两个子节点中较小的节点小于该节点，就将他们交换，知道两个子节点都比该元素大为止。 就上面二叉堆而言，删除的元素为元素1。

第一步：删掉元素1，元素6空出来，如下：

<img src="http://image.tinx.top/image-20211201105813424.png" alt="image-20211201105813424" style="zoom:50%;float:left;" />

第二步：与其两个子节点（元素2、元素3）比较，都小，将其中较小的元素（元素2）放入到该空穴中:

<img src="http://image.tinx.top/image-20211201105845434.png" alt="image-20211201105845434" style="zoom:50%;float:left;" />

第三步：继续比较两个子节点（元素5、元素7），还是都小，则将较小的元素（元素5）放入到该空穴中：

<img src="http://image.tinx.top/image-20211201105922674.png" alt="image-20211201105922674" style="zoom:50%;float:left;" />

第四步：比较其子节点（元素8），比该节点小，则元素6放入该空穴位置不会影响二叉堆的树结构，放入：

<img src="http://image.tinx.top/image-20211201111922340.png" alt="image-20211201111922340" style="zoom:50%;float:left;" />

## DelayQueue

> DelayQueue实现的关键主要有如下几个：
>
> 1. 可重入锁ReentrantLock
> 2. 用于阻塞和通知的Condition对象
> 3. 根据Delay时间排序的优先级队列：PriorityQueue
> 4. 用于优化阻塞通知的线程元素leader
>
> ReentrantLock、Condition这两个对象就不需要阐述了，他是实现整个BlockingQueue的核心。PriorityQueue是一个支持优先级线程排序的队列

DelayQueue是一个支持延时操作的无界阻塞队列。列头的元素是最先“到期”的元素，如果队列里面没有元素到期，是不能从列头获取元素的，哪怕有元素也不行。也就是说只有在延迟期满时才能够从队列中去元素。

它主要运用于如下场景：

缓存系统的设计：缓存是有一定的时效性的，可以用DelayQueue保存缓存的有效期，然后利用一个线程查询DelayQueue，如果取到元素就证明该缓存已经失效了。
定时任务的调度：DelayQueue保存当天将要执行的任务和执行时间，一旦取到元素（任务），就执行该任务。
DelayQueue采用支持优先级的PriorityQueue来实现，但是队列中的元素必须要实现Delayed接口，Delayed接口用来标记那些应该在给定延迟时间之后执行的对象，该接口提供了getDelay()方法返回元素节点的剩余时间。同时，元素也必须要实现compareTo()方法，compareTo()方法需要提供与getDelay()方法一致的排序。

## SynchronousQueue

> 1. SynchronousQueue没有容量。与其他BlockingQueue不同，SynchronousQueue是一个不存储元素的BlockingQueue。每一个put操作必须要等待一个take操作，否则不能继续添加元素，反之亦然。
> 2. 因为没有容量，所以对应 peek, contains, clear, isEmpty ... 等方法其实是无效的。例如clear是不执行任何操作的，contains始终返回false,peek始终返回null。
> 3. SynchronousQueue分为公平和非公平，默认情况下采用非公平性访问策略，当然也可以通过构造函数来设置为公平性访问策略（为true即可）。
> 4. 若使用 TransferQueue, 则队列中永远会存在一个 dummy node（这点后面详细阐述）。

SynchronousQueue是一个神奇的队列，他是一个不存储元素的阻塞队列，也就是说他的每一个put操作都需要等待一个take操作，否则就不能继续添加元素了，有点儿像Exchanger，类似于生产者和消费者进行交换。

队列本身不存储任何元素，所以非常适用于传递性场景，两者直接进行对接。其吞吐量会高于ArrayBlockingQueue和LinkedBlockingQueue。

SynchronousQueue支持公平和非公平的访问策略，在默认情况下采用非公平性，也可以通过构造函数来设置为公平性。

SynchronousQueue的实现核心为Transferer接口，该接口有TransferQueue和TransferStack两个实现类，分别对应着公平策略和非公平策略。接口Transferer有一个tranfer()方法，该方法定义了转移数据，如果e != null，相当于将一个数据交给消费者，如果e == null，则相当于从一个生产者接收一个消费者交出的数据。

## LinkedTransferQueue

> LinkedTransferQueue是基于链表的FIFO无界阻塞队列
>
> LinkedTransferQueue采用一种**预占模式**。什么意思呢？有就直接拿走，没有就占着这个位置直到拿到或者超时或者中断。即消费者线程到队列中取元素时，如果发现队列为空，则会生成一个null节点，然后park住等待生产者。后面如果生产者线程入队时发现有一个null元素节点，这时生产者就不会入列了，直接将元素填充到该节点上，唤醒该节点的线程，被唤醒的消费者线程拿东西走人。是不是有点儿[SynchronousQueue](http://cmsblogs.com/?p=2418)的味道？

LinkedTransferQueue是一个由链表组成的的无界阻塞队列，该队列是一个相当牛逼的队列：它是ConcurrentLinkedQueue、SynchronousQueue (公平模式下)、无界的LinkedBlockingQueues等的超集。

与其他BlockingQueue相比，他多实现了一个接口TransferQueue，该接口是对BlockingQueue的一种补充，多了tryTranfer()和transfer()两类方法：

tranfer()：若当前存在一个正在等待获取的消费者线程，即立刻移交之。 否则，会插入当前元素e到队列尾部，并且等待进入阻塞状态，到有消费者线程取走该元素
tryTranfer()： 若当前存在一个正在等待获取的消费者线程（使用take()或者poll()函数），使用该方法会即刻转移/传输对象元素e；若不存在，则返回false，并且不进入队列。这是一个不阻塞的操作

## LinkedBlockingDeque

> LinkedBlockingDeque 继承AbstractQueue，实现接口BlockingDeque，而BlockingDeque又继承接口BlockingQueue，BlockingDeque是支持两个附加操作的 Queue，这两个操作是：获取元素时等待双端队列变为非空；存储元素时等待双端队列中的空间变得可用。这两类操作就为LinkedBlockingDeque 的双向操作Queue提供了可能。BlockingDeque接口提供了一系列的以First和Last结尾的方法，如addFirst、addLast、peekFirst、peekLast。

前面的BlockingQueue都是单向的FIFO队列，而LinkedBlockingDeque则是一个由链表组成的双向阻塞队列，双向队列就意味着可以从对头、对尾两端插入和移除元素，同样意味着LinkedBlockingDeque支持FIFO、FILO两种操作方式。 

LinkedBlockingDeque是可选容量的，在初始化时可以设置容量防止其过度膨胀，如果不设置，默认容量大小为Integer.MAX_VALUE。另外双向阻塞队列可以运用在“工作窃取”模式中。

# Copy-On-Write容器

> 集合在我们开发中是使用得非常多的，包括在并发环境，我们知道Map接口在高并发时可以使用ConcurrentHashMap，但是List、Set貌似没有相应的ConcurrentList、ConcurrentSet，那么怎么解决List、Set高并发环境下的使用呢？

第一种使用锁，也就是在get、remove、add等操作时增加Lock锁，如下：

```java
add(E e){
	lock.lock();
	// add操作
	lock.unlock();
}

get(int i ){
	lock.lock();
	// get操作
	lock.unlock();
}
```

这样虽然保证了线程的安全性，但是由于是所有操作都是同一个锁，那么吞吐量势必就不是想象中的那么好了，怎么做呢？按照读写锁的思想，读写分离。CopyOnWrite容器就是这样一个读写分离的并发容器，他可以在非常多的并发场景中使用，目前JDK中有CopyOnWriteArrayList和CopyOnWriteSet两种。

## CopyOnWrite容器

Copy-On-Write即写时复制，简称COW。其思想是当我们往一个容器中添加元素时，不是直接新增，而是对当前容器进行Copy，复制出一份一模一样的新容器，然后往新容器中添加元素，添加成功后，再将原容器的引用指向新的容器，在读的时候仍然是读原容器，替换引用后就改为读新容器了。这样做的一个好处就是读写分离了，读的使用可以使用并发读并且是不需要加锁的。当然这种读写分离的思想是读和写分别在不同的容器。

### **add(E e) **

```java
public boolean add(E e) {
    final ReentrantLock lock = this.lock;
    // 获取锁
    lock.lock();
    try {
        // 数组
        Object[] elements = getArray();
        int len = elements.length;
        // 复制出新数组
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        // 新增元素e
        newElements[len] = e;
        // 修改引用
        setArray(newElements);
        return true;
    } finally {
        // 释放锁
        lock.unlock();
    }
}
```

### **get(int index)**

```java
public E get(int index) {
    return get(getArray(), index);
}
```

get操作相对于add来说，太简单了，就是a[index]，主要是读的时候是读得就容器里面的数据不会存在数据被修改的问题，所以直接根据数组下标获取即可。

虽然CopyOnWrite容器采用读写分离的思路避免了线程安全的问题，但是它仍然存在一些缺陷：内存占用和数据不一致问题。

内存占用：因为在写的时候采用复制的思想，那么写的时候内存里面会同时驻扎两个数组对象的内存。如果对象占用内存空间较大，那么可能会引发频繁的Yong GC和Full GC。

数据不一致问题：因为时效性的问题，写的时候读不一定可以里面读取到数据，例如你add(1)，那么在get(1)的时候不一定能够得到，所以CopyOnWrite容器只能保证数据的最终一致性。

最后，CopyOnWrite容器适用于读多写少的并发场景，如果写太频繁了则会导致大量数据复制操作，这会严重影响系统的性能。

# ThreadLocal

> 该类提供了线程局部 (thread-local) 变量。这些变量不同于它们的普通对应物，因为访问某个变量（通过其get 或 set 方法）的每个线程都有自己的局部变量，它独立于变量的初始化副本。ThreadLocal实例通常是类中的 private static 字段，它们希望将状态与某一个线程（例如，用户 ID 或事务 ID）相关联。

所以ThreadLocal与线程同步机制不同，线程同步机制是多个线程共享同一个变量，而ThreadLocal是为每一个线程创建一个单独的变量副本，故而每个线程都可以独立地改变自己所拥有的变量副本，而不会影响其他线程所对应的副本。可以说ThreadLocal为多线程环境下变量问题提供了另外一种解决思路。

ThreadLocal定义了四个方法：

get()：返回此线程局部变量的当前线程副本中的值。
initialValue()：返回此线程局部变量的当前线程的“初始值”。
remove()：移除此线程局部变量当前线程的值。
set(T value)：将此线程局部变量的当前线程副本中的值设置为指定值。
除了这四个方法，ThreadLocal内部还有一个静态内部类ThreadLocalMap，该内部类才是实现线程隔离机制的关键，get()、set()、remove()都是基于该内部类操作。ThreadLocalMap提供了一种用键值对方式存储每一个线程的变量副本的方法，key为当前ThreadLocal对象，value则是对应线程的变量副本。

对于ThreadLocal需要注意的有两点：

ThreadLocal实例本身是不存储值，它只是提供了一个在当前线程中找到副本值得key。
是ThreadLocal包含在Thread中，而不是Thread包含在ThreadLocal中，有些小伙伴会弄错他们的关系。

下图是Thread、ThreadLocal、ThreadLocalMap的关系:

![image-20211130152620938](http://image.tinx.top/image-20211130152620938.png)

## ThreadLocal源码解析

ThreadLocal虽然解决了这个多线程变量的复杂问题，但是它的源码实现却是比较简单的。ThreadLocalMap是实现ThreadLocal的关键，我们先从它入手。

### ThreadLocalMap

ThreadLocalMap其内部利用Entry来实现key-value的存储，如下：

```java
static class Entry extends WeakReference<ThreadLocal<?>> {
    /** The value associated with this ThreadLocal. */
    Object value;

    Entry(ThreadLocal<?> k, Object v) {
        super(k);
        value = v;
    }
}
```

从上面代码中可以看出Entry的key就是ThreadLocal，而value就是值。同时，Entry也继承WeakReference，所以说Entry所对应key（ThreadLocal实例）的引用为一个弱引用。

ThreadLocalMap的源码稍微多了点，我们就看两个最核心的方法getEntry()、set(ThreadLocal<?> key, Object value)方法。

#### **set(ThreadLocal<?> key, Object value)**

```java
private void set(ThreadLocal<?> key, Object value) {
    ThreadLocal.ThreadLocalMap.Entry[] tab = table;
    int len = tab.length;

    // 根据 ThreadLocal 的散列值，查找对应元素在数组中的位置
    int i = key.threadLocalHashCode & (len-1);

    // 采用“线性探测法”，寻找合适位置
    for (ThreadLocal.ThreadLocalMap.Entry e = tab[i];
        e != null;
        e = tab[i = nextIndex(i, len)]) {
        ThreadLocal<?> k = e.get();

        // key 存在，直接覆盖
        if (k == key) {
            e.value = value;
            return;
        }

        // key == null，但是存在值（因为此处的e != null），说明之前的ThreadLocal对象已经被回收了
        if (k == null) {
            // 用新元素替换陈旧的元素
            replaceStaleEntry(key, value, i);
            return;
        }
    }

    // ThreadLocal对应的key实例不存在也没有陈旧元素，new 一个
    tab[i] = new ThreadLocal.ThreadLocalMap.Entry(key, value);

    int sz = ++size;

    // cleanSomeSlots 清楚陈旧的Entry（key == null）
    // 如果没有清理陈旧的 Entry 并且数组中的元素大于了阈值，则进行 rehash
    if (!cleanSomeSlots(i, sz) && sz >= threshold)
        rehash();
}
```

这个set()操作和我们在集合了解的put()方式有点儿不一样，虽然他们都是key-value结构，不同在于他们解决散列冲突的方式不同。集合Map的put()采用的是拉链法，而ThreadLocalMap的set()则是采用开放定址法.

set()操作除了存储元素外，还有一个很重要的作用，就是replaceStaleEntry()和cleanSomeSlots()，这两个方法可以清除掉key == null 的实例，防止内存泄漏。在set()方法中还有一个变量很重要：threadLocalHashCode，定义如下：

```java
private final int threadLocalHashCode = nextHashCode();
```

从名字上面我们可以看出threadLocalHashCode应该是ThreadLocal的散列值，定义为final，表示ThreadLocal一旦创建其散列值就已经确定了，生成过程则是调用nextHashCode()：

```java
private static AtomicInteger nextHashCode = new AtomicInteger();

private static final int HASH_INCREMENT = 0x61c88647;

private static int nextHashCode() {
    return nextHashCode.getAndAdd(HASH_INCREMENT);
}
```

nextHashCode表示分配下一个ThreadLocal实例的threadLocalHashCode的值，HASH_INCREMENT则表示分配两个ThradLocal实例的threadLocalHashCode的增量，从nextHashCode就可以看出他们的定义。

#### **getEntry()**

```java
private Entry getEntry(ThreadLocal<?> key) {
    int i = key.threadLocalHashCode & (table.length - 1);
    Entry e = table[i];
    if (e != null && e.get() == key)
        return e;
    else
        return getEntryAfterMiss(key, i, e);
}
```

由于采用了开放定址法，所以当前key的散列值和元素在数组的索引并不是完全对应的，首先取一个探测数（key的散列值），如果所对应的key就是我们所要找的元素，则返回，否则调用getEntryAfterMiss()，如下：

```java
private Entry getEntryAfterMiss(ThreadLocal<?> key, int i, Entry e) {
    Entry[] tab = table;
    int len = tab.length;

    while (e != null) {
        ThreadLocal<?> k = e.get();
        if (k == key)
            return e;
        if (k == null)
            expungeStaleEntry(i);
        else
            i = nextIndex(i, len);
        e = tab[i];
    }
    return null;
}
```

这里有一个重要的地方，当key == null时，调用了expungeStaleEntry()方法，该方法用于处理key == null，有利于GC回收，能够有效地避免内存泄漏。

### get()

> 返回当前线程所对应的线程变量

```java
public T get() {
    // 获取当前线程
    Thread t = Thread.currentThread();

    // 获取当前线程的成员变量 threadLocal
    ThreadLocalMap map = getMap(t);
    if (map != null) {
        // 从当前线程的ThreadLocalMap获取相对应的Entry
        ThreadLocalMap.Entry e = map.getEntry(this);
        if (e != null) {
            @SuppressWarnings("unchecked")

            // 获取目标值        
            T result = (T)e.value;
            return result;
        }
    }
    return setInitialValue();
}
```

首先通过当前线程获取所对应的成员变量ThreadLocalMap，然后通过ThreadLocalMap获取当前ThreadLocal的Entry，最后通过所获取的Entry获取目标值result。

getMap()方法可以获取当前线程所对应的ThreadLocalMap，如下：

```java
ThreadLocalMap getMap(Thread t) {
    return t.threadLocals;
}
```

### set(T value)

> 设置当前线程的线程局部变量的值。

```java
public void set(T value) {
    Thread t = Thread.currentThread();
    ThreadLocalMap map = getMap(t);
    if (map != null)
        map.set(this, value);
    else
        createMap(t, value);
}
```

获取当前线程所对应的ThreadLocalMap，如果不为空，则调用ThreadLocalMap的set()方法，key就是当前ThreadLocal，如果不存在，则调用createMap()方法新建一个，如下：

```java
void createMap(Thread t, T firstValue) {
    t.threadLocals = new ThreadLocalMap(this, firstValue);
}
```

### initialValue()

> 返回该线程局部变量的初始值。

```java
protected T initialValue() {
    return null;
}
```

该方法定义为protected级别且返回为null，很明显是要子类实现它的，所以我们在使用ThreadLocal的时候一般都应该覆盖该方法。该方法不能显示调用，只有在第一次调用get()或者set()方法时才会被执行，并且仅执行1次。

### remove()

> 将当前线程局部变量的值删除。

```java
public void remove() {
    ThreadLocalMap m = getMap(Thread.currentThread());
    if (m != null)
        m.remove(this);
}
```

该方法的目的是减少内存的占用。当然，我们不需要显示调用该方法，因为一个线程结束后，它所对应的局部变量就会被垃圾回收。

## ThreadLocal为什么会内存泄漏

前面提到每个Thread都有一个ThreadLocal.ThreadLocalMap的map，该map的key为ThreadLocal实例，它为一个弱引用，我们知道弱引用有利于GC回收。当ThreadLocal的key == null时，GC就会回收这部分空间，但是value却不一定能够被回收，因为他还与Current Thread存在一个强引用关系，如下

![image-20211130153821025](http://image.tinx.top/image-20211130153821025.png)

由于存在这个强引用关系，会导致value无法回收。如果这个线程对象不会销毁那么这个强引用关系则会一直存在，就会出现内存泄漏情况。所以说只要这个线程对象能够及时被GC回收，就不会出现内存泄漏。如果碰到线程池，那就更坑了。

那么要怎么避免这个问题呢？

在前面提过，在ThreadLocalMap中的setEntry()、getEntry()，如果遇到key == null的情况，会对value设置为null。当然我们也可以显示调用ThreadLocal的remove()方法进行处理。

下面再对ThreadLocal进行简单的总结：

> + ThreadLocal 不是用于解决共享变量的问题的，也不是为了协调线程同步而存在，而是为了方便每个线程处理自己的状态而引入的一个机制。这点至关重要。
> + 每个Thread内部都有一个ThreadLocal.ThreadLocalMap类型的成员变量，该成员变量用来存储实际的ThreadLocal变量副本。
> + ThreadLocal并不是为线程保存对象的副本，它仅仅只起到一个索引的作用。它的主要木得视为每一个线程隔离一个类的实例，这个实例的作用范围仅限于线程内部。

# 线程池

## Executor

<img src="http://image.tinx.top/image-20211130154130435.png" alt="image-20211130154130435" style="zoom:50%;" />

Executor，任务的执行者，线程池框架中几乎所有类都直接或者间接实现Executor接口，它是线程池框架的基础。Executor提供了一种将“任务提交”与“任务执行”分离开来的机制，它仅提供了一个Execute()方法用来执行已经提交的Runnable任务。

```java
public interface Executor {
    void execute(Runnable command);
}
```

### **ExcutorService**

继承Executor，它是“执行者服务”接口，它是为"执行者接口Executor"服务而存在的。准确的地说，ExecutorService提供了"将任务提交给执行者的接口(submit方法)"，"让执行者执行任务(invokeAll, invokeAny方法)"的接口等等。

```java
public interface ExecutorService extends Executor {

    /**
	 * 启动一次顺序关闭，执行以前提交的任务，但不接受新任务
     */
    void shutdown();

    /**
     * 试图停止所有正在执行的活动任务，暂停处理正在等待的任务，并返回等待执行的任务列表
     */
    List<Runnable> shutdownNow();

    /**
     * 如果此执行程序已关闭，则返回 true。
     */
    boolean isShutdown();

    /**
     * 如果关闭后所有任务都已完成，则返回 true
     */
    boolean isTerminated();

    /**
     * 请求关闭、发生超时或者当前线程中断，无论哪一个首先发生之后，都将导致阻塞，直到所有任务完成执行
     */
    boolean awaitTermination(long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 提交一个返回值的任务用于执行，返回一个表示任务的未决结果的 Future
     */
    <T> Future<T> submit(Callable<T> task);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    <T> Future<T> submit(Runnable task, T result);

    /**
     * 提交一个 Runnable 任务用于执行，并返回一个表示该任务的 Future
     */
    Future<?> submit(Runnable task);

    /**
     * 执行给定的任务，当所有任务完成时，返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks)
        throws InterruptedException;

    /**
     * 执行给定的任务，当所有任务完成或超时期满时（无论哪个首先发生），返回保持任务状态和结果的 Future 列表
     */
    <T> List<Future<T>> invokeAll(Collection<? extends Callable<T>> tasks,
                                  long timeout, TimeUnit unit)
        throws InterruptedException;

    /**
     * 执行给定的任务，如果某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks)
        throws InterruptedException, ExecutionException;

    /**
     * 执行给定的任务，如果在给定的超时期满前某个任务已成功完成（也就是未抛出异常），则返回其结果
     */
    <T> T invokeAny(Collection<? extends Callable<T>> tasks,
                    long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### **AbstractExecutorService**

抽象类，实现ExecutorService接口，为其提供默认实现。AbstractExecutorService除了实现ExecutorService接口外，还提供了newTaskFor()方法返回一个RunnableFuture，在运行的时候，它将调用底层可调用任务，作为 Future 任务，它将生成可调用的结果作为其结果，并为底层任务提供取消操作。

### **ScheduledExecutorService**

继承ExcutorService，为一个“延迟”和“定期执行”的ExecutorService。他提供了一些如下几个方法安排任务在给定的延时执行或者周期性执行。

```java
// 创建并执行在给定延迟后启用的 ScheduledFuture。
<V> ScheduledFuture<V> schedule(Callable<V> callable, long delay, TimeUnit unit)

// 创建并执行在给定延迟后启用的一次性操作。
ScheduledFuture<?> schedule(Runnable command, long delay, TimeUnit unit)

// 创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；
//也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。
ScheduledFuture<?> scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit)

// 创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。
ScheduledFuture<?> scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit)

```

### **Executors**

静态工厂类，提供了Executor、ExecutorService、ScheduledExecutorService、ThreadFactory 、Callable 等类的静态工厂方法，通过这些工厂方法我们可以得到相对应的对象。

1. 创建并返回设置有常用配置字符串的 ExecutorService 的方法。
2. 创建并返回设置有常用配置字符串的 ScheduledExecutorService 的方法。
3. 创建并返回“包装的”ExecutorService 方法，它通过使特定于实现的方法不可访问来禁用重新配置。
4. 创建并返回 ThreadFactory 的方法，它可将新创建的线程设置为已知的状态。
5. 创建并返回非闭包形式的 Callable 的方法，这样可将其用于需要 Callable 的执行方法中。

### Future

Future接口和实现Future接口的FutureTask代表了线程池的异步计算结果。

AbstractExecutorService提供了newTaskFor()方法返回一个RunnableFuture，除此之外当我们把一个Runnable或者Callable提交给（submit()）ThreadPoolExecutor或者ScheduledThreadPoolExecutor时，他们则会向我们返回一个FutureTask对象。如下：

```java
  protected <T> RunnableFuture<T> newTaskFor(Runnable runnable, T value) {
      return new FutureTask<T>(runnable, value);
  }

    protected <T> RunnableFuture<T> newTaskFor(Callable<T> callable) {
      return new FutureTask<T>(callable);
  }

<T> Future<T> submit(Callable<T> task)
<T> Future<T> submit(Runnable task, T result)
Future<> submit(Runnable task)
```

![image-20211130154504804](http://image.tinx.top/image-20211130154504804.png)

作为异步计算的顶层接口，Future对具体的Runnable或者Callable任务提供了三种操作：执行任务的取消、查询任务是否完成、获取任务的执行结果。其接口定义如下：

```java
public interface Future<V> {

    /**
     * 试图取消对此任务的执行
	 * 如果任务已完成、或已取消，或者由于某些其他原因而无法取消，则此尝试将失败。
	 * 当调用 cancel 时，如果调用成功，而此任务尚未启动，则此任务将永不运行。
	 * 如果任务已经启动，则 mayInterruptIfRunning 参数确定是否应该以试图停止任务的方式来中断执行此任务的线程
     */
    boolean cancel(boolean mayInterruptIfRunning);

    /**
     * 如果在任务正常完成前将其取消，则返回 true
     */
    boolean isCancelled();

    /**
     * 如果任务已完成，则返回 true
     */
    boolean isDone();

    /**
     *   如有必要，等待计算完成，然后获取其结果
     */
    V get() throws InterruptedException, ExecutionException;

    /**
     * 如有必要，最多等待为使计算完成所给定的时间之后，获取其结果（如果结果可用）
     */
    V get(long timeout, TimeUnit unit)
        throws InterruptedException, ExecutionException, TimeoutException;
}
```

### **RunnableFuture**

继承Future、Runnable两个接口，为两者的合体，即所谓的Runnable的Future。提供了一个run()方法可以完成Future并允许访问其结果。

```java
public interface RunnableFuture<V> extends Runnable, Future<V> {
	//在未被取消的情况下，将此 Future 设置为计算的结果
    void run();
}
```

### **FutureTask**

实现RunnableFuture接口，既可以作为Runnable被执行，也可以作为Future得到Callable的返回值。

### 线程池组

Executor框架提供了三种线程池，他们都可以通过工具类Executors来创建。

#### **FixedThreadPool**

```java
public static ExecutorService newFixedThreadPool(int nThreads) {
    return new ThreadPoolExecutor(nThreads, nThreads,
                                  0L, TimeUnit.MILLISECONDS,
                                  new LinkedBlockingQueue<Runnable>());
}
```

corePoolSize 和 maximumPoolSize都设置为创建FixedThreadPool时指定的参数nThreads，意味着当线程池满时且阻塞队列也已经满时，如果继续提交任务，则会直接走拒绝策略，该线程池不会再新建线程来执行任务，而是直接走拒绝策略。FixedThreadPool使用的是默认的拒绝策略，即AbortPolicy，则直接抛出异常。

keepAliveTime设置为0L，表示空闲的线程会立刻终止。

workQueue则是使用LinkedBlockingQueue，但是没有设置范围，那么则是最大值（Integer.MAX_VALUE），这基本就相当于一个无界队列了。使用该“无界队列”则会带来哪些影响呢？当线程池中的线程数量等于corePoolSize 时，如果继续提交任务，该任务会被添加到阻塞队列workQueue中，当阻塞队列也满了之后，则线程池会新建线程执行任务直到maximumPoolSize。由于FixedThreadPool使用的是“无界队列”LinkedBlockingQueue，那么maximumPoolSize参数无效，同时指定的拒绝策略AbortPolicy也将无效。而且该线程池也不会拒绝提交的任务，如果客户端提交任务的速度快于任务的执行，那么keepAliveTime也是一个无效参数。

其运行图如下

![image-20211130160926779](http://image.tinx.top/image-20211130160926779.png)

#### **SingleThreadExecutor**

SingleThreadExecutor是使用单个worker线程的Executor，定义如下：

```java
public static ExecutorService newSingleThreadExecutor() {
    return new FinalizableDelegatedExecutorService
        (new ThreadPoolExecutor(1, 1,
                                0L, TimeUnit.MILLISECONDS,
                                new LinkedBlockingQueue<Runnable>()));
}
```

作为单一worker线程的线程池，SingleThreadExecutor把corePool和maximumPoolSize均被设置为1，和FixedThreadPool一样使用的是无界队列LinkedBlockingQueue,所以带来的影响和FixedThreadPool一样。

![image-20211130161134990](http://image.tinx.top/image-20211130161134990.png)

#### **CachedThreadPool**

CachedThreadPool是一个会根据需要创建新线程的线程池 ，他定义如下：

```java
public static ExecutorService newCachedThreadPool() {
    return new ThreadPoolExecutor(0, Integer.MAX_VALUE,
                                  60L, TimeUnit.SECONDS,
                                  new SynchronousQueue<Runnable>());
}
```

CachedThreadPool的corePool为0，maximumPoolSize为Integer.MAX_VALUE，这就意味着所有的任务一提交就会加入到阻塞队列中。keepAliveTime这是为60L，unit设置为TimeUnit.SECONDS，意味着空闲线程等待新任务的最长时间为60秒，空闲线程超过60秒后将会被终止。

阻塞队列采用的SynchronousQueue，而我们知道SynchronousQueue是一个没有元素的阻塞队列，加上corePool = 0 ，maximumPoolSize = Integer.MAX_VALUE，这样就会存在一个问题，

***如果主线程提交任务的速度远远大于CachedThreadPool的处理速度，则CachedThreadPool会不断地创建新线程来执行任务，这样有可能会导致系统耗尽CPU和内存资源，所以在使用该线程池是，一定要注意控制并发的任务数，否则创建大量的线程可能导致严重的性能问题。***

![image-20211130161442777](http://image.tinx.top/image-20211130161442777.png)

#### 任务提交

线程池根据业务不同的需求提供了两种方式提交任务：Executor.execute()、ExecutorService.submit()。其中ExecutorService.submit()可以获取该任务执行的Future。
我们以Executor.execute()为例，来看看线程池的任务提交经历了那些过程。

定义：

```java
public interface Executor {

    void execute(Runnable command);
}
```

ThreadPoolExecutor提供实现：

```java
public void execute(Runnable command) {
    if (command == null)
        throw new NullPointerException();
    int c = ctl.get();
    if (workerCountOf(c) < corePoolSize) {
        if (addWorker(command, true))
            return;
        c = ctl.get();
    }
    if (isRunning(c) && workQueue.offer(command)) {
        int recheck = ctl.get();
        if (! isRunning(recheck) && remove(command))
            reject(command);
        else if (workerCountOf(recheck) == 0)
            addWorker(null, false);
    }
    else if (!addWorker(command, false))
        reject(command);
}
```

执行流程如下：

1. 如果线程池当前线程数小于corePoolSize，则调用addWorker创建新线程执行任务，成功返回true，失败执行步骤2。
2. 如果线程池处于RUNNING状态，则尝试加入阻塞队列，如果加入阻塞队列成功，则尝试进行Double Check，如果加入失败，则执行步骤3。
3. 如果线程池不是RUNNING状态或者加入阻塞队列失败，则尝试创建新线程直到maxPoolSize，如果失败，则调用reject()方法运行相应的拒绝策略。

在步骤2中如果加入阻塞队列成功了，则会进行一个Double Check的过程。Double Check过程的主要目的是判断加入到阻塞队里中的线程是否可以被执行。如果线程池不是RUNNING状态，则调用remove()方法从阻塞队列中删除该任务，然后调用reject()方法处理任务。否则需要确保还有线程执行。

##### **addWorker**

当线程中的当前线程数小于corePoolSize，则调用addWorker()创建新线程执行任务，当前线程数则是根据**ctl**变量来获取的，调用workerCountOf(ctl)获取低29位即可：

```java
private static int workerCountOf(int c)  { return c & CAPACITY; }
```

addWorker(Runnable firstTask, boolean core)方法用于创建线程执行任务，源码如下：

```java
private boolean addWorker(Runnable firstTask, boolean core) {
    retry:
    for (;;) {
        int c = ctl.get();

        // 获取当前线程状态
        int rs = runStateOf(c);


        if (rs >= SHUTDOWN &&
                ! (rs == SHUTDOWN &&
                        firstTask == null &&
                        ! workQueue.isEmpty()))
            return false;

        // 内层循环，worker + 1
        for (;;) {
            // 线程数量
            int wc = workerCountOf(c);
            // 如果当前线程数大于线程最大上限CAPACITY  return false
            // 若core == true，则与corePoolSize 比较，否则与maximumPoolSize ，大于 return false
            if (wc >= CAPACITY ||
                    wc >= (core ? corePoolSize : maximumPoolSize))
                return false;
            // worker + 1,成功跳出retry循环
            if (compareAndIncrementWorkerCount(c))
                break retry;

            // CAS add worker 失败，再次读取ctl
            c = ctl.get();

            // 如果状态不等于之前获取的state，跳出内层循环，继续去外层循环判断
            if (runStateOf(c) != rs)
                continue retry;
        }
    }

    boolean workerStarted = false;
    boolean workerAdded = false;
    Worker w = null;
    try {

        // 新建线程：Worker
        w = new Worker(firstTask);
        // 当前线程
        final Thread t = w.thread;
        if (t != null) {
            // 获取主锁：mainLock
            final ReentrantLock mainLock = this.mainLock;
            mainLock.lock();
            try {

                // 线程状态
                int rs = runStateOf(ctl.get());

                // rs < SHUTDOWN ==> 线程处于RUNNING状态
                // 或者线程处于SHUTDOWN状态，且firstTask == null（可能是workQueue中仍有未执行完成的任务，创建没有初始任务的worker线程执行）
                if (rs < SHUTDOWN ||
                        (rs == SHUTDOWN && firstTask == null)) {

                    // 当前线程已经启动，抛出异常
                    if (t.isAlive()) // precheck that t is startable
                        throw new IllegalThreadStateException();

                    // workers是一个HashSet<Worker>
                    workers.add(w);

                    // 设置最大的池大小largestPoolSize，workerAdded设置为true
                    int s = workers.size();
                    if (s > largestPoolSize)
                        largestPoolSize = s;
                    workerAdded = true;
                }
            } finally {
                // 释放锁
                mainLock.unlock();
            }
            // 启动线程
            if (workerAdded) {
                t.start();
                workerStarted = true;
            }
        }
    } finally {

        // 线程启动失败
        if (! workerStarted)
            addWorkerFailed(w);
    }
    return workerStarted;
}
```

1. 判断当前线程是否可以添加任务，如果可以则进行下一步，否则return false；
2. rs >= SHUTDOWN ，表示当前线程处于SHUTDOWN ，STOP、TIDYING、TERMINATED状态
3. rs == SHUTDOWN , firstTask != null时不允许添加线程，因为线程处于SHUTDOWN 状态，不允许添加任务
4. rs == SHUTDOWN , firstTask == null，但workQueue.isEmpty() == true，不允许添加线程，因为firstTask == null是为了添加一个没有任务的线程然后再从workQueue中获取任务的，如果workQueue == null，则说明添加的任务没有任何意义。
5. 内嵌循环，通过CAS worker + 1
6. 获取主锁mailLock，如果线程池处于RUNNING状态获取处于SHUTDOWN状态且 firstTask == null，则将任务添加到workers Queue中，然后释放主锁mainLock，然后启动线程，然后return true，如果中途失败导致workerStarted= false，则调用addWorkerFailed()方法进行处理。

在这里需要好好理论addWorker中的参数，在execute()方法中，有三处调用了该方法：

+ 第一次：==workerCountOf(c) < corePoolSize > addWorker(command, true)==，这个很好理解，当然线程池的线程数量小于 corePoolSize ，则新建线程执行任务即可，在执行过程core == true，内部与corePoolSize比较即可。
+ 第二次：加入阻塞队列进行Double Check时，==else if (workerCountOf(recheck) == 0) ==>addWorker(null, false)==。如果线程池中的线程0，按照道理应该该任务应该新建线程执行任务，但是由于已经该任务已经添加到了阻塞队列，那么就在线程池中新建一个空线程，然后从阻塞队列中取线程即可。
  第三次：线程池不是RUNNING状态或者加入阻塞队列失败：==else if (!addWorker(command, false))==，这里core == fase，则意味着是与maximumPoolSize比较。

在新建线程执行任务时，将讲Runnable包装成一个Worker，Woker为ThreadPoolExecutor的内部类

##### **Woker内部类**

```java
private final class Worker extends AbstractQueuedSynchronizer
        implements Runnable {
    private static final long serialVersionUID = 6138294804551838833L;

    // task 的thread
    final Thread thread;

    // 运行的任务task
    Runnable firstTask;

    volatile long completedTasks;

    Worker(Runnable firstTask) {

        //设置AQS的同步状态private volatile int state，是一个计数器，大于0代表锁已经被获取
        setState(-1);
        this.firstTask = firstTask;

        // 利用ThreadFactory和 Worker这个Runnable创建的线程对象
        this.thread = getThreadFactory().newThread(this);
    }

    // 任务执行
    public void run() {
        runWorker(this);
    }

}
```

从Worker的源码中我们可以看到Woker继承AQS，实现Runnable接口，所以可以认为Worker既是一个可以执行的任务，也可以达到获取锁释放锁的效果。这里继承AQS主要是为了方便线程的中断处理。这里注意两个地方：构造函数、run()。构造函数主要是做三件事：1.设置同步状态state为-1，同步状态大于0表示就已经获取了锁，2.设置将当前任务task设置为firstTask，3.利用Worker本身对象this和ThreadFactory创建线程对象。

当线程thread启动（调用start()方法）时，其实就是执行Worker的run()方法，内部调用runWorker()。

##### **runWorker**

```java
final void runWorker(Worker w) {

    // 当前线程
    Thread wt = Thread.currentThread();

    // 要执行的任务
    Runnable task = w.firstTask;

    w.firstTask = null;

    // 释放锁，运行中断
    w.unlock(); // allow interrupts
    boolean completedAbruptly = true;
    try {
        while (task != null || (task = getTask()) != null) {
            // worker 获取锁
            w.lock();

            // 确保只有当线程是stoping时，才会被设置为中断，否则清楚中断标示
            // 如果线程池状态 >= STOP ,且当前线程没有设置中断状态，则wt.interrupt()
            // 如果线程池状态 < STOP，但是线程已经中断了，再次判断线程池是否 >= STOP，如果是 wt.interrupt()
            if ((runStateAtLeast(ctl.get(), STOP) ||
                    (Thread.interrupted() &&
                            runStateAtLeast(ctl.get(), STOP))) &&
                    !wt.isInterrupted())
                wt.interrupt();
            try {
                // 自定义方法
                beforeExecute(wt, task);
                Throwable thrown = null;
                try {
                    // 执行任务
                    task.run();
                } catch (RuntimeException x) {
                    thrown = x; throw x;
                } catch (Error x) {
                    thrown = x; throw x;
                } catch (Throwable x) {
                    thrown = x; throw new Error(x);
                } finally {
                    afterExecute(task, thrown);
                }
            } finally {
                task = null;
                // 完成任务数 + 1
                w.completedTasks++;
                // 释放锁
                w.unlock();
            }
        }
        completedAbruptly = false;
    } finally {
        processWorkerExit(w, completedAbruptly);
    }
}
```

运行流程

1. 根据worker获取要执行的任务task，然后调用unlock()方法释放锁，这里释放锁的主要目的在于中断，因为在new Worker时，设置的state为-1，调用unlock()方法可以将state设置为0，这里主要原因就在于interruptWorkers()方法只有在state >= 0时才会执行；
2. 通过getTask()获取执行的任务，调用task.run()执行，当然在执行之前会调用worker.lock()上锁，执行之后调用worker.unlock()放锁；
3. 在任务执行前后，可以根据业务场景自定义beforeExecute() 和 afterExecute()方法，则两个方法在ThreadPoolExecutor中是空实现；
4. 如果线程执行完成，则会调用getTask()方法从阻塞队列中获取新任务，如果阻塞队列为空，则根据是否超时来判断是否需要阻塞；
5. task == null或者抛出异常（beforeExecute()、task.run()、afterExecute()均有可能）导致worker线程终止，则调用processWorkerExit()方法处理worker退出流程。

##### **getTask()**

```java
private Runnable getTask() {
    boolean timedOut = false; // Did the last poll() time out?

    for (;;) {

        // 线程池状态
        int c = ctl.get();
        int rs = runStateOf(c);

        // 线程池中状态 >= STOP 或者 线程池状态 == SHUTDOWN且阻塞队列为空，则worker - 1，return null
        if (rs >= SHUTDOWN && (rs >= STOP || workQueue.isEmpty())) {
            decrementWorkerCount();
            return null;
        }

        int wc = workerCountOf(c);

        // 判断是否需要超时控制
        boolean timed = allowCoreThreadTimeOut || wc > corePoolSize;

        if ((wc > maximumPoolSize || (timed && timedOut)) && (wc > 1 || workQueue.isEmpty())) {
            if (compareAndDecrementWorkerCount(c))
                return null;
            continue;
        }

        try {

            // 从阻塞队列中获取task
            // 如果需要超时控制，则调用poll()，否则调用take()
            Runnable r = timed ?
                    workQueue.poll(keepAliveTime, TimeUnit.NANOSECONDS) :
                    workQueue.take();
            if (r != null)
                return r;
            timedOut = true;
        } catch (InterruptedException retry) {
            timedOut = false;
        }
    }
}
```

timed == true，调用poll()方法，如果在keepAliveTime时间内还没有获取task的话，则返回null，继续循环。timed == false，则调用take()方法，该方法为一个阻塞方法，没有任务时会一直阻塞挂起，直到有任务加入时对该线程唤醒，返回任务。

在runWorker()方法中，无论最终结果如何，都会执行processWorkerExit()方法对worker进行退出处理。

##### **processWorkerExit()**

```java
private void processWorkerExit(Worker w, boolean completedAbruptly) {
    // true：用户线程运行异常,需要扣减
    // false：getTask方法中扣减线程数量
    if (completedAbruptly)
        decrementWorkerCount();

    // 获取主锁
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        completedTaskCount += w.completedTasks;
        // 从HashSet中移出worker
        workers.remove(w);
    } finally {
        mainLock.unlock();
    }

    // 有worker线程移除，可能是最后一个线程退出需要尝试终止线程池
    tryTerminate();

    int c = ctl.get();
    // 如果线程为running或shutdown状态，即tryTerminate()没有成功终止线程池，则判断是否有必要一个worker
    if (runStateLessThan(c, STOP)) {
        // 正常退出，计算min：需要维护的最小线程数量
        if (!completedAbruptly) {
            // allowCoreThreadTimeOut 默认false：是否需要维持核心线程的数量
            int min = allowCoreThreadTimeOut ? 0 : corePoolSize;
            // 如果min ==0 或者workerQueue为空，min = 1
            if (min == 0 && ! workQueue.isEmpty())
                min = 1;

            // 如果线程数量大于最少数量min，直接返回，不需要新增线程
            if (workerCountOf(c) >= min)
                return; // replacement not needed
        }
        // 添加一个没有firstTask的worker
        addWorker(null, false);
    }
}
```

首先completedAbruptly的值来判断是否需要对线程数-1处理，如果completedAbruptly == true，说明在任务运行过程中出现了异常，那么需要进行减1处理，否则不需要，因为减1处理在getTask()方法中处理了。然后从HashSet中移出该worker，过程需要获取mainlock。然后调用tryTerminate()方法处理，该方法是对最后一个线程退出做终止线程池动作。如果线程池没有终止，那么线程池需要保持一定数量的线程，则通过addWorker(null,false)新增一个空的线程。

##### **addWorkerFailed()**

在addWorker()方法中，如果线程t==null，或者在add过程出现异常，会导致workerStarted == false，那么在最后会调用addWorkerFailed()方法：

```java
private void addWorkerFailed(Worker w) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        // 从HashSet中移除该worker
        if (w != null)
            workers.remove(w);

        // 线程数 - 1
        decrementWorkerCount();
        // 尝试终止线程
        tryTerminate();
    } finally {
        mainLock.unlock();
    }
}
```

##### **tryTerminate()**

当线程池涉及到要移除worker时候都会调用tryTerminate()，该方法主要用于判断线程池中的线程是否已经全部移除了，如果是的话则关闭线程池。

```java
final void tryTerminate() {
    for (;;) {
        int c = ctl.get();
        // 线程池处于Running状态
        // 线程池已经终止了
        // 线程池处于ShutDown状态，但是阻塞队列不为空
        if (isRunning(c) ||
                runStateAtLeast(c, TIDYING) ||
                (runStateOf(c) == SHUTDOWN && ! workQueue.isEmpty()))
            return;

        // 执行到这里，就意味着线程池要么处于STOP状态，要么处于SHUTDOWN且阻塞队列为空
        // 这时如果线程池中还存在线程，则会尝试中断线程
        if (workerCountOf(c) != 0) {
            // /线程池还有线程，但是队列没有任务了，需要中断唤醒等待任务的线程
            // （runwoker的时候首先就通过w.unlock设置线程可中断，getTask最后面的catch处理中断）
            interruptIdleWorkers(ONLY_ONE);
            return;
        }

        final ReentrantLock mainLock = this.mainLock;
        mainLock.lock();
        try {
            // 尝试终止线程池
            if (ctl.compareAndSet(c, ctlOf(TIDYING, 0))) {
                try {
                    terminated();
                } finally {
                    // 线程池状态转为TERMINATED
                    ctl.set(ctlOf(TERMINATED, 0));
                    termination.signalAll();
                }
                return;
            }
        } finally {
            mainLock.unlock();
        }
    }
}
```

在关闭线程池的过程中，如果线程池处于STOP状态或者处于SHUDOWN状态且阻塞队列为null，则线程池会调用interruptIdleWorkers()方法中断所有线程，注意ONLY_ONE== true，表示仅中断一个线程。

##### **interruptIdleWorkers**

```java
private void interruptIdleWorkers(boolean onlyOne) {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers) {
            Thread t = w.thread;
            if (!t.isInterrupted() && w.tryLock()) {
                try {
                    t.interrupt();
                } catch (SecurityException ignore) {
                } finally {
                    w.unlock();
                }
            }
            if (onlyOne)
                break;
        }
    } finally {
        mainLock.unlock();
    }
}
```

onlyOne==true仅终止一个线程，否则终止所有线程。

#### 线程终止

线程池ThreadPoolExecutor提供了shutdown()和shutDownNow()用于关闭线程池。

shutdown()：按过去执行已提交任务的顺序发起一个有序的关闭，但是不接受新任务。

shutdownNow() :尝试停止所有的活动执行任务、暂停等待任务的处理，并返回等待执行的任务列表。

##### **shutdown**

```java
public void shutdown() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        // 推进线程状态
        advanceRunState(SHUTDOWN);
        // 中断空闲的线程
        interruptIdleWorkers();
        // 交给子类实现
        onShutdown();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
}
```

##### **shutdownNow**

```java
public List<Runnable> shutdownNow() {
    List<Runnable> tasks;
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        checkShutdownAccess();
        advanceRunState(STOP);
        // 中断所有线程
        interruptWorkers();
        // 返回等待执行的任务列表
        tasks = drainQueue();
    } finally {
        mainLock.unlock();
    }
    tryTerminate();
    return tasks;
}
```

与shutdown不同，shutdownNow会调用interruptWorkers()方法中断所有线程。

```java
private void interruptWorkers() {
    final ReentrantLock mainLock = this.mainLock;
    mainLock.lock();
    try {
        for (Worker w : workers)
            w.interruptIfStarted();
    } finally {
        mainLock.unlock();
    }
}
```

同时会调用drainQueue()方法返回等待执行到任务列表。

```java
private List<Runnable> drainQueue() {
    BlockingQueue<Runnable> q = workQueue;
    ArrayList<Runnable> taskList = new ArrayList<Runnable>();
    q.drainTo(taskList);
    if (!q.isEmpty()) {
        for (Runnable r : q.toArray(new Runnable[0])) {
            if (q.remove(r))
                taskList.add(r);
        }
    }
    return taskList;
}
```

## ThreadPoolExecutor

### 内部状态

线程有五种状态：新建，就绪，运行，阻塞，死亡，线程池同样有五种状态：Running, SHUTDOWN, STOP, TIDYING, TERMINATED。

```java
private final AtomicInteger ctl = new AtomicInteger(ctlOf(RUNNING, 0));
private static final int COUNT_BITS = Integer.SIZE - 3;
private static final int CAPACITY   = (1 << COUNT_BITS) - 1;

// runState is stored in the high-order bits
private static final int RUNNING    = -1 << COUNT_BITS;
private static final int SHUTDOWN   =  0 << COUNT_BITS;
private static final int STOP       =  1 << COUNT_BITS;
private static final int TIDYING    =  2 << COUNT_BITS;
private static final int TERMINATED =  3 << COUNT_BITS;

// Packing and unpacking ctl
private static int runStateOf(int c)     { return c & ~CAPACITY; }
private static int workerCountOf(int c)  { return c & CAPACITY; }
private static int ctlOf(int rs, int wc) { return rs | wc; }
```

变量**ctl**定义为AtomicInteger ，其功能非常强大，记录了“线程池中的任务数量”和“线程池的状态”两个信息。共32位，其中高3位表示"线程池状态"，低29位表示"线程池中的任务数量"。

```sql
RUNNING            -- 对应的高3位值是111。
SHUTDOWN       -- 对应的高3位值是000。
STOP                   -- 对应的高3位值是001。
TIDYING              -- 对应的高3位值是010。
TERMINATED     -- 对应的高3位值是011。
```

RUNNING：处于RUNNING状态的线程池能够接受新任务，以及对新添加的任务进行处理。

SHUTDOWN：处于SHUTDOWN状态的线程池不可以接受新任务，但是可以对已添加的任务进行处理。

STOP：处于STOP状态的线程池不接收新任务，不处理已添加的任务，并且会中断正在处理的任务。

TIDYING：当所有的任务已终止，ctl记录的"任务数量"为0，线程池会变为TIDYING状态。当线程池变为TIDYING状态时，会执行钩子函数terminated()。terminated()在ThreadPoolExecutor类中是空的，若用户想在线程池变为TIDYING时，进行相应的处理；可以通过重载terminated()函数来实现。

TERMINATED：线程池彻底终止的状态。

各个状态的转换如下：

![image-20211130160348141](http://image.tinx.top/image-20211130160348141.png)

### 创建线程池

我们可以通过ThreadPoolExecutor构造函数来创建一个线程池：

```java
public ThreadPoolExecutor(int corePoolSize,
                          int maximumPoolSize,
                          long keepAliveTime,
                          TimeUnit unit,
                          BlockingQueue<Runnable> workQueue,
                          ThreadFactory threadFactory,
                          RejectedExecutionHandler handler) {
    if (corePoolSize < 0 ||
        maximumPoolSize <= 0 ||
        maximumPoolSize < corePoolSize ||
        keepAliveTime < 0)
        throw new IllegalArgumentException();
    if (workQueue == null || threadFactory == null || handler == null)
        throw new NullPointerException();
    this.corePoolSize = corePoolSize;
    this.maximumPoolSize = maximumPoolSize;
    this.workQueue = workQueue;
    this.keepAliveTime = unit.toNanos(keepAliveTime);
    this.threadFactory = threadFactory;
    this.handler = handler;
}
```

共有七个参数，每个参数含义如下：

+ corePoolSize:

> 线程池中核心线程的数量。当提交一个任务时，线程池会新建一个线程来执行任务，直到当前线程数等于corePoolSize。如果调用了线程池的prestartAllCoreThreads()方法，线程池会提前创建并启动所有基本线程。

+ maximumPoolSize

> 线程池中允许的最大线程数。线程池的阻塞队列满了之后，如果还有任务提交，如果当前的线程数小于maximumPoolSize，则会新建线程来执行任务。注意，如果使用的是无界队列，该参数也就没有什么效果了。

+ keepAliveTime

> 线程空闲的时间。线程的创建和销毁是需要代价的。线程执行完任务后不会立即销毁，而是继续存活一段时间：keepAliveTime。默认情况下，该参数只有在线程数大于corePoolSize时才会生效。

+ unit

>  keepAliveTime的单位。TimeUnit

+ workQueue

> 用来保存等待执行的任务的阻塞队列，等待的任务必须实现Runnable接口。我们可以选择如下几种：
>
> ArrayBlockingQueue：基于数组结构的有界阻塞队列，FIFO。
> LinkedBlockingQueue：基于链表结构的有界阻塞队列，FIFO。
> SynchronousQueue：不存储元素的阻塞队列，每个插入操作都必须等待一个移出操作，反之亦然。【
> PriorityBlockingQueue：具有优先界别的阻塞队列。

### **threadFactory**

用于设置创建线程的工厂。该对象可以通过Executors.defaultThreadFactory()，如下：

```java
public static ThreadFactory defaultThreadFactory() {
    return new DefaultThreadFactory();
}
```

返回的是DefaultThreadFactory对象，源码如下：

```java
static class DefaultThreadFactory implements ThreadFactory {
    private static final AtomicInteger poolNumber = new AtomicInteger(1);
    private final ThreadGroup group;
    private final AtomicInteger threadNumber = new AtomicInteger(1);
    private final String namePrefix;

    DefaultThreadFactory() {
        SecurityManager s = System.getSecurityManager();
        group = (s != null) ? s.getThreadGroup() :
                              Thread.currentThread().getThreadGroup();
        namePrefix = "pool-" +
                      poolNumber.getAndIncrement() +
                     "-thread-";
    }

    public Thread newThread(Runnable r) {
        Thread t = new Thread(group, r,
                              namePrefix + threadNumber.getAndIncrement(),
                              0);
        if (t.isDaemon())
            t.setDaemon(false);
        if (t.getPriority() != Thread.NORM_PRIORITY)
            t.setPriority(Thread.NORM_PRIORITY);
        return t;
    }
}
```

ThreadFactory的左右就是提供创建线程的功能的线程工厂。他是通过newThread()方法提供创建线程的功能，newThread()方法创建的线程都是“非守护线程”而且“线程优先级都是Thread.NORM_PRIORITY”。

### **handler**

RejectedExecutionHandler，线程池的拒绝策略。所谓拒绝策略，是指将任务添加到线程池中时，线程池拒绝该任务所采取的相应策略。当向线程池中提交任务时，如果此时线程池中的线程已经饱和了，而且阻塞队列也已经满了，则线程池会选择一种拒绝策略来处理该任务。

线程池提供了四种拒绝策略：

1. AbortPolicy：直接抛出异常，默认策略；
2. CallerRunsPolicy：用调用者所在的线程来执行任务；
3. DiscardOldestPolicy：丢弃阻塞队列中靠最前的任务，并执行当前任务；
4. DiscardPolicy：直接丢弃任务；

当然我们也可以实现自己的拒绝策略，例如记录日志等等，实现RejectedExecutionHandler接口即可。

## ScheduledThreadPoolExecutor

我们知道Timer与TimerTask虽然可以实现线程的周期和延迟调度，但是Timer与TimerTask存在一些缺陷，所以对于这种定期、周期执行任务的调度策略，我们一般都是推荐ScheduledThreadPoolExecutor来实现。下面就深入分析ScheduledThreadPoolExecutor是如何来实现线程的周期、延迟调度的。

ScheduledThreadPoolExecutor，继承ThreadPoolExecutor且实现了ScheduledExecutorService接口，它就相当于提供了“延迟”和“周期执行”功能的ThreadPoolExecutor。在JDK API中是这样定义它的：ThreadPoolExecutor，它可另行安排在给定的延迟后运行命令，或者定期执行命令。需要多个辅助线程时，或者要求 ThreadPoolExecutor 具有额外的灵活性或功能时，此类要优于 Timer。 一旦启用已延迟的任务就执行它，但是有关何时启用，启用后何时执行则没有任何实时保证。按照提交的先进先出 (FIFO) 顺序来启用那些被安排在同一执行时间的任务。

它提供了四个构造方法：

```java
public ScheduledThreadPoolExecutor(int corePoolSize) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue());
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), threadFactory);
}

public ScheduledThreadPoolExecutor(int corePoolSize,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), handler);
}


public ScheduledThreadPoolExecutor(int corePoolSize,
                                   ThreadFactory threadFactory,
                                   RejectedExecutionHandler handler) {
    super(corePoolSize, Integer.MAX_VALUE, 0, NANOSECONDS,
            new DelayedWorkQueue(), threadFactory, handler);
}
```

在ScheduledThreadPoolExecutor的构造函数中，我们发现它都是利用ThreadLocalExecutor来构造的，唯一变动的地方就在于它所使用的阻塞队列变成了DelayedWorkQueue，而不是ThreadLocalhExecutor的LinkedBlockingQueue（通过Executors产生ThreadLocalhExecutor对象）。DelayedWorkQueue为ScheduledThreadPoolExecutor中的内部类，它其实和阻塞队列DelayQueue有点儿类似。DelayQueue是可以提供延迟的阻塞队列，它只有在延迟期满时才能从中提取元素，其列头是延迟期满后保存时间最长的Delayed元素。如果延迟都还没有期满，则队列没有头部，并且 poll 将返回 null。所以DelayedWorkQueue中的任务必然是按照延迟时间从短到长来进行排序的。

ScheduledThreadPoolExecutor提供了如下四个方法，也就是四个调度器：

1. schedule(Callable callable, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的 ScheduledFuture
2. schedule(Runnable command, long delay, TimeUnit unit) :创建并执行在给定延迟后启用的一次性操作。
3. scheduleAtFixedRate(Runnable command, long initialDelay, long period, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，后续操作具有给定的周期；也就是将在 initialDelay 后开始执行，然后在 initialDelay+period 后执行，接着在 initialDelay + 2 * period 后执行，依此类推。
4. scheduleWithFixedDelay(Runnable command, long initialDelay, long delay, TimeUnit unit) :创建并执行一个在给定初始延迟后首次启用的定期操作，随后，在每一次执行终止和下一次执行开始之间都存在给定的延迟。

查看着四个方法的源码，会发现其实他们的处理逻辑都差不多，所以我们就挑scheduleWithFixedDelay方法来分析，如下：

```java
public ScheduledFuture<?> scheduleWithFixedDelay(Runnable command,
                                                   long initialDelay,
                                                   long delay,
                                                   TimeUnit unit) {
      if (command == null || unit == null)
          throw new NullPointerException();
      if (delay <= 0)
          throw new IllegalArgumentException();
      ScheduledFutureTask<Void> sft =
          new ScheduledFutureTask<Void>(command,
                                        null,
                                        triggerTime(initialDelay, unit),
                                        unit.toNanos(-delay));
      RunnableScheduledFuture<Void> t = decorateTask(command, sft);
      sft.outerTask = t;
      delayedExecute(t);
      return t;
}
```

scheduleWithFixedDelay方法处理的逻辑如下：

1. 校验，如果参数不合法则抛出异常
2. 构造一个task，该task为ScheduledFutureTask
3. 调用delayedExecute()方法做后续相关处理

这段代码涉及两个类ScheduledFutureTask和RunnableScheduledFuture，其中RunnableScheduledFuture不用多说，他继承RunnableFuture和ScheduledFuture两个接口，除了具备RunnableFuture和ScheduledFuture两类特性外，它还定义了一个方法isPeriodic() ，该方法用于判断执行的任务是否为定期任务，如果是则返回true。而ScheduledFutureTask作为ScheduledThreadPoolExecutor的内部类，它扮演着极其重要的作用，因为它的作用则是负责ScheduledThreadPoolExecutor中任务的调度。

ScheduledFutureTask内部继承FutureTask，实现RunnableScheduledFuture接口，它内部定义了三个比较重要的变量

```java
/** 任务被添加到ScheduledThreadPoolExecutor中的序号 */
private final long sequenceNumber;
/** 任务要执行的具体时间 */
private long time;
/**  任务的间隔周期 */
private final long period;
```

这三个变量与任务的执行有着非常密切的关系，什么关系？先看ScheduledFutureTask的几个构造函数和核心方法：

```java
ScheduledFutureTask(Runnable r, V result, long ns) {
    super(r, result);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Runnable r, V result, long ns, long period) {
    super(r, result);
    this.time = ns;
    this.period = period;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Callable<V> callable, long ns) {
    super(callable);
    this.time = ns;
    this.period = 0;
    this.sequenceNumber = sequencer.getAndIncrement();
}

ScheduledFutureTask(Callable<V> callable, long ns) {
        super(callable);
        this.time = ns;
        this.period = 0;
        this.sequenceNumber = sequencer.getAndIncrement();
}
```

ScheduledFutureTask 提供了四个构造方法，这些构造方法与上面三个参数是不是一一对应了？这些参数有什么用，如何用，则要看ScheduledFutureTask在那些方法使用了该方法，在ScheduledFutureTask中有一个compareTo()方法：

```java
public int compareTo(Delayed other) {
    if (other == this) // compare zero if same object
        return 0;
    if (other instanceof ScheduledFutureTask) {
        ScheduledFutureTask<?> x = (ScheduledFutureTask<?>)other;
        long diff = time - x.time;
        if (diff < 0)
            return -1;
        else if (diff > 0)
            return 1;
        else if (sequenceNumber < x.sequenceNumber)
            return -1;
        else
            return 1;
    }
    long diff = getDelay(NANOSECONDS) - other.getDelay(NANOSECONDS);
    return (diff < 0) ? -1 : (diff > 0) ? 1 : 0;
}
```

提供一个排序算法，该算法规则是：首先按照time排序，time小的排在前面，大的排在后面，如果time相同，则使用sequenceNumber排序，小的排在前面，大的排在后面。那么为什么在这个类里面提供compareTo()方法呢？在前面就介绍过ScheduledThreadPoolExecutor在构造方法中提供的是DelayedWorkQueue()队列中，也就是说ScheduledThreadPoolExecutor是把任务添加到DelayedWorkQueue中的，而DelayedWorkQueue则是类似于DelayQueue，内部维护着一个以时间为先后顺序的队列，所以compareTo()方法使用与DelayedWorkQueue队列对其元素ScheduledThreadPoolExecutor task进行排序的算法。

排序已经解决了，那么ScheduledThreadPoolExecutor 是如何对task任务进行调度和延迟的呢？任何线程的执行，都是通过run()方法执行，ScheduledThreadPoolExecutor 的run()方法如下：

```java
public void run() {
    boolean periodic = isPeriodic();
    if (!canRunInCurrentRunState(periodic))
        cancel(false);
    else if (!periodic)
        ScheduledFutureTask.super.run();
    else if (ScheduledFutureTask.super.runAndReset()) {
        setNextRunTime();
        reExecutePeriodic(outerTask);
    }
}
```

1. 调用isPeriodic()获取该线程是否为周期性任务标志，然后调用canRunInCurrentRunState()方法判断该线程是否可以执行，如果不可以执行则调用cancel()取消任务。
2. 如果当线程已经到达了执行点，则调用run()方法执行task，该run()方法是在FutureTask中定义的。
3. 否则调用runAndReset()方法运行并充值，调用setNextRunTime()方法计算任务下次的执行时间，重新把任务添加到队列中，让该任务可以重复执行。

### **isPeriodic()**

该方法用于判断指定的任务是否为定期任务。

```java
public boolean isPeriodic() {
    return period != 0;
}
```

canRunInCurrentRunState()判断任务是否可以取消，cancel()取消任务，这两个方法比较简单，而run()执行任务，runAndReset()运行并重置状态，牵涉比较广，我们放在FutureTask后面介绍。所以重点介绍setNextRunTime()和reExecutePeriodic()这两个涉及到延迟的方法。

### **setNextRunTime()**

setNextRunTime()方法用于重新计算任务的下次执行时间。如下：

```java
private void setNextRunTime() {
    long p = period;
    if (p > 0)
        time += p;
    else
        time = triggerTime(-p);
}
```

该方法定义很简单，p > 0 ,time += p ，否则调用triggerTime()方法重新计算time：

```java
long triggerTime(long delay) {
    return now() +
        ((delay < (Long.MAX_VALUE >> 1)) ? delay : overflowFree(delay));
}
```

### **reExecutePeriodic**

```java
void reExecutePeriodic(RunnableScheduledFuture<?> task) {
    if (canRunInCurrentRunState(true)) {
        super.getQueue().add(task);
        if (!canRunInCurrentRunState(true) && remove(task))
            task.cancel(false);
        else
            ensurePrestart();
    }
}
```

reExecutePeriodic重要的是调用super.getQueue().add(task);将任务task加入的队列DelayedWorkQueue中
