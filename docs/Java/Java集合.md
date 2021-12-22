## Java框架体系

![](http://image.tinx.top/20211115133739.png)

​	

## List集合

![](http://image.tinx.top/20211115133855.png)

### ArrayList

#### 基础特性

+ <font color=red>ArrayList 可以添加多个null </font>
+ 底层是以数组来实现数据存储的
+ ArrayList基本等同于Vector，差别就是ArrayList 线程不安全，Vector线程安全

#### 扩容机制

![](http://image.tinx.top/20211115140209.png)

![image-20211115140443236](/Users/wills/Library/Application Support/typora-user-images/image-20211115140443236.png)

+ ArrayList中维护了一个Object类型的数组
+ 当创建ArrayList对象时，使用的是无参构造器，则初始elementData容量为0，第一次添加，它的初始容量为10，如果再次扩容，则扩大到之前的1.5倍
+ 如果使用了有参构造器，则初始的elemenetData的容量为参数的容量，如果要再次扩容，则扩大到之前的1.5倍

#### 步骤

如果是无参构造方法：

1. 创建了一个空的elementData数组

2. 当要添加元素的时候，add源码是：

   ![](http://image.tinx.top/20210629161225.png)

   1. 先确保数组中剩余的容量可以扩容

      ![](http://image.tinx.top/20210629161652.png)

      先检查是不是默认0长度的数组的，如果是就是找出默认容量和最小容量的最大值(因为后面会扩大minCapacity)

      ![](http://image.tinx.top/20210629162227.png)

      其中modCount++ 是记录是否是 多线程更改这个容量，如果是多线程更改容量，会抛出错误

      后面是判断是否容量足够，不足够调用grow方法

      ![](http://image.tinx.top/20210629162924.png)

      步骤是：

      	1. 先获取当前数组的长度
      	2. 创建一个newCapacity 将之前的长度扩大至1.5倍(位运算，不了解可以去补习一下位运算相关知识)
      	3. 判断当前newCapacity是否比最小容量小，如果是，就将最小容量赋值给newCapacity
      	4. 判断当前newCapacity是否比最大容量大，如果是，调用hugeCapacity
      	5. 复制之前的数组给新的数组
      
   2. 然后将要添加的值放到数组的最后一个元素中
   
   3. 返回true

### CopyOnWriteArrayList

> 这是一个线程安全的集合类，如果你是在多线程条件下使用集合类，推荐使用CopyOnWriteArrayList
>
> 他的实现原理跟ArrayList差不多，底层都是一个Object数组，唯一的不同就是他比ArrayList多了一把ReentrantLock锁，在添加时会锁住

![](http://image.tinx.top/20211116145610.png)

#### 添加时的代码讲解

```java
// 源码：
public boolean add(E e) {
  // 在添加时会调用类中声明的这把锁进行锁住，在添加完成时finally代码中进行unlock操作
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] elements = getArray();
        int len = elements.length;
        Object[] newElements = Arrays.copyOf(elements, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```



### Vector

> 代码几乎和ArrayList相同，就是相关的像INIT_CAPACITY等都是直接写死的，略微的不同而已

#### 底层原理

+ 底层和ArrayList一样是一个 ```protect Object[] elementData```

+ Vector 底层是线程同步的，里面的方法都带有synchornized

  ![](http://image.tinx.top/20210702093911.png)

+ 在开发中，需要线程安全的单列集合时，考虑使用Vector(但是性能太差，可以考虑CopyOnWriteArrayList 里面实现了Lock锁，性能更高，更优选择)

+ 如果创建时使用了无参，在扩容时会自动扩容至之前的两倍，如果是有参的话，会在参数上 * 2 作为扩容

  ![](http://image.tinx.top/20210702093801.png)

### LinkedList

#### 特性

+ 底层实现了双向链表 、 双端队列的特点

  ![](http://image.tinx.top/20211115150303.png)

+ 底层维护了两个属性  ```first``` 和 ```last``` 分别指向 首节点 和 尾节点,里面又维护了 ```prev```、```next```、```item```三个属性，用来指向 之前一个元素 之后一个元素 当前元素，最终实现双向链表

  ![](http://image.tinx.top/20211115150527.png)

+ 可以添加任意元素(元素可以重复)，包括null

+ 线程不安全，没有实现线程同步 synchornized

### ArrayList 和 LinkedList 比较

|            | 底层结构                     | 增删效率           | 改查效率 |
| :--------: | ---------------------------- | ------------------ | -------- |
| ArrayList  | 可变数组                     | 较低，数组复制扩容 | 较高     |
| LinkedList | 双向链表(正向next，逆向prev) | 较高，通过链表追加 | 较低     |

#### 如何选择ArrayList 和 LinkedList?

1. 如果我们改查的操作较多，选择ArrayList
2. 如果我们增删的操作比较多，选择LinkedList
3. 一般来说，在程序中，80%-90%都是查询，因此大部分情况下会选择ArrayList

### 补充内容:RandomAccess接口

```
public interface RandomAccess {
}
```

查看源码我们发现实际上 `RandomAccess` 接口中什么都没有定义。所以，在我看来 `RandomAccess` 接口不过是一个标识罢了。标识什么？ 标识实现这个接口的类具有随机访问功能。

在 `binarySearch（`）方法中，它要判断传入的list 是否 `RamdomAccess` 的实例，如果是，调用`indexedBinarySearch（）`方法，如果不是，那么调用`iteratorBinarySearch（）`方法

```
    public static <T>
    int binarySearch(List<? extends Comparable<? super T>> list, T key) {
        if (list instanceof RandomAccess || list.size()<BINARYSEARCH_THRESHOLD)
            return Collections.indexedBinarySearch(list, key);
        else
            return Collections.iteratorBinarySearch(list, key);
    }
```

`ArrayList` 实现了 `RandomAccess` 接口， 而 `LinkedList` 没有实现。为什么呢？我觉得还是和底层数据结构有关！`ArrayList` 底层是数组，而 `LinkedList` 底层是链表。数组天然支持随机访问，时间复杂度为 O（1），所以称为快速随机访问。链表需要遍历到特定位置才能访问特定位置的元素，时间复杂度为 O（n），所以不支持快速随机访问。，`ArrayList` 实现了 `RandomAccess` 接口，就表明了他具有快速随机访问功能。 `RandomAccess` 接口只是标识，并不是说 `ArrayList` 实现 `RandomAccess` 接口才具有快速随机访问功能的！

下面再总结一下 list 的遍历方式选择：

- 实现了 `RandomAccess` 接口的list，优先选择普通 for 循环 ，其次 foreach,
- 未实现 `RandomAccess`接口的list，优先选择iterator遍历（foreach遍历底层也是通过iterator实现的,），大size的数据，千万不要使用普通for循环

## Set集合

+ Set接口的常用方法

  和List接口一样，Set接口也是Collection的子接口，因此常用方法和Collection接口一样

+ Set接口的遍历方式

  同Collection的遍历方式一样，因为Set接口是Collection接口的子接口

  1. 可以使用迭代器
  2. 增强for
  3. <font color=red>不能使用索引的方式来获取</font>

+ 存储

  1. 可以存储null
  2. 不能存放重复元素
  3. 存储的数据是无序的，取出的顺序的顺序虽然不是添加的顺序，但是也是固定的

### HashSet

> 什么是hash?
>
> hash也称为散列、哈希，基本原理就是把任意长度的输入，通过Hash算法变成固定长度的输出。这个映射规则就是对应的hash算法，而原始数据映射后的二进制串就是哈希值
>
> 1. hash不能反向推导出原始的数据
> 2. 输入数据微小的变化会得到完全不同的hash值，相同的会得到相同的值
> 3. 哈希算法的执行效率要更加高效，长的文本也可以快速得出hash值
> 4. hash算法的<font color=red>冲突概率</font>较小
>
> 由于Hash原理就是将输入空间的值映射成hash空间内，而hash的值空间远远小于输入的空间。
>
> 根据抽屉原理，一定会存在不同的输入被映射成相同的输出的情况。
>
> 抽屉原理：
>
> ​	> 桌子上有10个苹果， 要把这10个苹果放到9个抽屉里，无论怎么放，我们会发现至少有一个抽屉会放不少于2个苹果

#### 特性

1. 实现了Set接口

2. HashSet底层是HashMap（jdk1.7是 数组+链表 jdk1.8是数组 + 链表 + 红黑树）
   
   ![](http://image.tinx.top/20211115152936.png)
   
   1. 第一次添加时，table数组扩容到16，临界值(threshold)是 16 * 加载因子(loadFactor)是 0.75 = 12
   
      ![](http://image.tinx.top/20211115154114.png)
   
   2. 如果 table 数组使用到了临界值12，就会扩容到 16 * 2 =32，新的临界值就是 32 * 0.75 = 24，以此类推
   
   3. 在Java8中，如果一条链表的元素个数超过 TREEIFY_THRESHOLD(默认是8)，并且table的大小 >= MIN_TREEIFY_CAPACITY(默认64)，就会进行树化(红黑树)

![](http://image.tinx.top/20210709102855.png)

3. 可以存放null值，但是只能有一个Null
4. HashSet不保证元素是有序的，取决于hash后，再确定索引的结果。(即，不保证存放元素的顺序和取出顺序一致，如果要有顺序的可以考虑实现了SortedSet接口的TreeSet)
5. 不能有重复元素(根据hashCode和equals方法来判断的)



#### Add

##### 步骤

1. HashSet底层是HashMap
2. 添加一个元素时，先得到hash值，会转换成 -> 索引值
3. 找到存储数据表table，看这个索引位置是否已经存放有元素
4. 如果没有，直接加入
5. 如果有，则调用equals比较，如果相同，就放弃添加，如果不相同，则添加到最后
6. 在Java8中，如果一条链表的元素个数超过 TREEIFY_THRESHOLD(默认是8)，并且table的大小 >= MIN_TREEIFY_CAPACITY(默认64)，就会进行树化(红黑树)



#### LinkedHashSet

+ 是HashSet的子类

  ![](http://image.tinx.top/20211115160118.png)

+ LinkedHashSet 底层是 一个 LinkedHashMap

  ![](http://image.tinx.top/20211115160050.png)

  ![](http://image.tinx.top/20211115160230.png)

+ LinkedHashSet 根据元素 HashCode来决定元素的存储位置，同时使用链表维护元素的次序，这使元素看起来是以插入的顺序的保存的

+ LinkedHashSet不允许插入重复元素

### TreeSet

> 自动排序的Set不可重复集合，底层是TreeMap

我们在创建时可以向构造函数中创建一个比较器 new Compartor() .... 用来比较集合中的元素，达到排序的作用

### CopyOnWriteArraySet

> 本类是一个线程安全的没有重复元素的Set，他跟上面的Set不同，此类不依赖上面的Map结构，而是直接依赖了 CopyOnWriteArrayList，他的间接数据结构就是个数组！（面试可能会问）

![](http://image.tinx.top/20211116150350.png)

#### 添加方法

```java
// 源码： 直接调用 CopyOnWriteArrayList的addIfAbsent(E e)方法
public boolean add(E e) {
    return al.addIfAbsent(e);
}

// 这一步查看的是是否已经存在这个元素了
public boolean addIfAbsent(E e) {
    Object[] snapshot = getArray();
    return indexOf(e, snapshot, 0, snapshot.length) >= 0 ? false :
        addIfAbsent(e, snapshot);
}

// 最终还是调用了 依赖的 CopyOnWriteArrayList中的 addIfAbsent
private boolean addIfAbsent(E e, Object[] snapshot) {
  	// 加锁
    final ReentrantLock lock = this.lock;
    lock.lock();
    try {
        Object[] current = getArray();
        int len = current.length;
        if (snapshot != current) {
            // Optimize for lost race to another addXXX operation
            int common = Math.min(snapshot.length, len);
            for (int i = 0; i < common; i++)
                if (current[i] != snapshot[i] && eq(e, current[i]))
                    return false;
            if (indexOf(e, current, common, len) >= 0)
                    return false;
        }
        Object[] newElements = Arrays.copyOf(current, len + 1);
        newElements[len] = e;
        setArray(newElements);
        return true;
    } finally {
        lock.unlock();
    }
}
```



## Map

![](http://image.tinx.top/20211115133830.png)

1. Map用于保存具有映射关系的数据： Key - Value

2. Map中的key和 value可以使任何引用类型的数据，会封装到HashMap$Node对象中(实现了Map.Entry接口)

   ![](http://image.tinx.top/20211115161114.png)

3. Map中的key不允许重复，原因和<font color=red>HashSet一样(key hash后的结果)</font>，如果重复就会覆盖原值

4. Map中的value可以重复

5. Map的key可以为null，value可以为Null，注意key和null只能有一个，value为Null可以有多个

6. 常用String类作为Map的key

7. key和value之间存在单向一对一的关系，即通过指定的key总能找到对应的value。



### HashMap

#### 扩容机制(和HashSet相同)

1. HashMap底层维护了Node类型的数组table，默认为Null

   ![](http://image.tinx.top/20211115161821.png)

2. 当创建对象时，将加载因子(loadfactor)初始化为0.75

   ![](http://image.tinx.top/20211115161958.png)

3. 当添加 key-val时，通过key的哈希值得到在table的索引
   1. 然后判断这个索引处出否有元素，如果没有元素就直接添加。
   2. 如果该索引处有元素没继续判断该元素的key和准备加入的key是否相等
      1. 如果相等，则直接替换val
      2. 如果不相等需要判断树结构还是链表结构，做出相应的处理。如果添加时发现容量不够，则需要扩容。

4. 第一次添加，则需要扩容的table初始容量为16，临界值为 (16(初始容量) *0.75(增长因子) = 12)

5. 以后再扩容，就需要扩容table容量为原来的2倍，临界值为原来的两倍，以此类推

6. 在Java8中，如果一条链表的元素>=TREEIFY_THRESHOLD（默认为8），并且table的大小 >= MIN_TREEIFY_CAPACITY(默认64)，就会进行转换红黑树的操作

   ![](http://image.tinx.top/20211115162615.png)

​		![](http://image.tinx.top/20211115162733.png)

​		然后如果是调用了resize，会在代码中，查看是否可以从树化转化为数组化，代码如下：		![](http://image.tinx.top/20211115163152.png)

![](http://image.tinx.top/20211115163013.png)		

#### Put流程

![](https://img-blog.csdnimg.cn/20210315155420212.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3NodW1veWlu,size_16,color_FFFFFF,t_70#pic_center)



##### Put源码

> (1) 首先获取put值的key的hashCode，然后调用putVal函数，向其中添加值
> (2) 在putVal函数中，首先判断是不是刚初始化的HashMap，如果是，就进行扩容操作 
> (3) 然后判断hashMap的数组中是否存在这个要插入的值的链表，如果不存在为Null，就创建一个新的结点(调用newNode函数)
> (4) 如果是存在的情况，就判断当前链表元素是否和传入的key相同，如果相同就，等待后续处理，将p赋值给e(详情看上面源码)
> (5) 如果不相同，就判断这个链表是否是树形结构，是的话，就调用putTreeVal函数，创建树的新节点
> (6) 否则，就遍历这个数组元素指向的这一条链表，遍历方法的内容就是，如果这是链表的最后一个元素(链表.next == null的情况)就会对这个链表添加新的结点(调用newNode方法)，然后再判断这个链表数量是否大于阈值等于(TREEIFY_THRESHOLD = 8) - 1 = 7，如果是就会再次判断查看这个hashMap的数组是否大于64，如果符合条件，就进行红黑树化，然后如果遍历的这个链表和要加入的值相同，就终止循环，说明重复值，不允许加入。
> (7) 如果旧值为null，并且唯一性标志符(onlyIfAbsent)为true，就进行值覆盖的操作
> (8) 然后最终在检查一下容量是否大于阈值，是的话要进行扩容，然后调用一下Map接口留下的钩子扩充函数afterNodeInsertion，返回null
> 补充： 为什么返回Null，因为在HashSet(底层是HashMap)调用这个putVal函数的时候，会最终判断是否添加成功的标志就是使用了三目运算符，判断是否 == null。
> // HashSet的add函数
> public boolean add(E e) {
>         return map.put(e, PRESENT)==null;
>  }

```java
public V put(K key, V value) {
    return putVal(hash(key), key, value, false, true);
}

// hash 函数
static final int hash(Object key) {
    int h;
  	// 调用这个对象的 hashCode方法，拿到这个类的hashCode 然后将这个 hashCode 位运算右移 16位(防止Hash碰撞)
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

```java
// 关键的设置值步骤，适用于hashSet hashMap等
final V putVal(int hash, K key, V value, boolean onlyIfAbsent, boolean evict) {
  // tab: 复制的table数组值
  // p: 拿到存储这个key的内容
  // n: hashMap 数组的长度
  // i: 指向复制的
    Node<K,V>[] tab; Node<K,V> p; int n, i;
  
  // 这个table指的是 hashMap中存储数据的Node<K,V>[]数组
  // 如果是这个刚被初始化出来，就会进行进行resize方法
  // final Node<K,V>[] resize() {
  //    Node<K,V>[] oldTab = table;
  //    int oldCap = (oldTab == null) ? 0 : oldTab.length; }
    if ((tab = table) == null || (n = tab.length) == 0)
        n = (tab = resize()).length;
  
  // 如果通过长度 - 1 和 hash 与位运算 得到的值为 null，那么就会创建一个 新的Node值,他的next为 null，作为链表的第一个
    if ((p = tab[i = (n - 1) & hash]) == null)
        tab[i] = newNode(hash, key, value, null);
    else {
        Node<K,V> e; K k;
      // 如果长度 -1 和 hash进行与位运算得到的值不为空，并且当前的值和传入要添加的值hash后相同
        if (p.hash == hash && ((k = p.key) == key || (key != null && key.equals(k))))
          // 将这个 将当前的值赋值给 e
            e = p;
      // 如果是红黑树 树形结构，会将这个值添加,然后将返回的树形结构赋值给 e
        else if (p instanceof TreeNode)
            e = ((TreeNode<K,V>)p).putTreeVal(this, tab, hash, key, value);
        else {
          	// 遍历这个桶链表
            for (int binCount = 0; ; ++binCount) {
              	// 如果当前链表遍历的下一个对象是 null
                if ((e = p.next) == null) {
                  // 将这个值放入桶链表当前值的下一个 链表node中
                    p.next = newNode(hash, key, value, null);
                  	// 如果 桶链表数量 大于等于 树化阈值(8) - 1 = 7
                    if (binCount >= TREEIFY_THRESHOLD - 1) // -1 for 1st
                      // 在这个函数中会判断当前数组数量是否大于等于64 如果大于等于的话会进行红黑树化这个链表
                        treeifyBin(tab, hash);
                  // 终止循环
                    break;
                }
              	// 如果遍历的这个链表和要加入的值相同，就终止循环，说明重复值，不允许加入
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    break;
              	// 否则就将这个值放入这个链表中
                p = e;
            }
        }
        if (e != null) { // existing mapping for key
            V oldValue = e.value;
            if (!onlyIfAbsent || oldValue == null)
                e.value = value;
            afterNodeAccess(e);
            return oldValue;
        }
    }
  	// 修改次数 + 1
    ++modCount;
  	// 如果下次添加的长度 大于 阈值，就会进行 扩充
    if (++size > threshold)
        resize();
  	// Map接口调用的方法，用于实现类的后续处理
    afterNodeInsertion(evict);
  	// 如果返回的是 null，后面会判断是否添加的元素是否 == null，如果是返回true，如果不是(上面有return),就会返回false表示添加失败
    return null;
}
```

##### hashcode方法中为什么要最高位要右移16位呢？

```java
// hash 函数
static final int hash(Object key) {
    int h;
  	// 调用这个对象的 hashCode方法，拿到这个类的hashCode 然后将这个 hashCode 位运算右移 16位(防止Hash碰撞)
    return (key == null) ? 0 : (h = key.hashCode()) ^ (h >>> 16);
}
```

> 举个例子，假设HashMap容量为16(二进制为10000，n-1=1111)，hashCode足够大
>
> A:00101010010001001011010100101011
>
> B:00101010010001101011010100101011
>
> HashMap的散列算法为(n-1)&hash，当n=16时，n-1只有后四位值为1，也就是真正参加运算的只有四位(n-1)&A = 1011,(n-1)&B=1011，明显不够分散。
>
> (h = key.hashCode()) ^ (h >>> 16)作用就是尽量让所有位参与运算，已知int大小为32位，(h = key.hashCode()) ^ (h >>> 16)可以让hashCode内部所有位先参与一遍亦或运算(高16位和低16位进行异或运算，可以更好的保留每一位的特征，看下列补充)
>
> A操作之后为：00101010010001001001111101101111
>
> B操作之后为：00101010010001101011111101101101
>
> 现在参与运算与运算的后四位就有区别了，就能散列到不同的桶中了。
>
> 补充：
>
> 亦或运算(^)：不同为1，相同为0  异或运算能更好的保留各部分的特征(变化的地方01都有)
>
> 与运算(&)：  有0则0，全1则1  结果偏向于0 (变化的地方都变0了)
>
> 或运算(|)：  有1则1，全0则0  结果偏向于1（变化的地方都变1了）

##### 为啥要进行(n-1)&hashcode？

让我们看一下hashMap是怎么存放元素的

```java
if ((tab = table) == null || (n = tab.length) == 0)
   n = (tab = resize()).length;
if ((p = tab[i = (n - 1) & hash]) == null)
   tab[i] = newNode(hash, key, value, null);
```

在这边jdk1.7和1.8是相同的，都是使用 n(table的容量与hashcode进行(&)与运算(位运算符，与运算：  有0则0，全1则1  结果偏向于0 (变化的地方都变0了)))

抛开上面提到的右移16位问题，你想想，一个对象取完hash值了，它得往entry数组里放吧，但是数组长度又是有限的，不可能一个萝卜一个坑。

以默认初始长度为16为例

想把一个编号大于16的数，在16个格子里找个位置放下，要是你自己设计会咋做呢(不能乱放)

绝大多数人都会选择取余法，就是：编号%格子数

小白举例： 17%16 = 1  放在第一个格子。

取余操作涉及到底层汇编和运算器电路设计，虽然我本科是电子信息的，但早就把知识还给老师了。

大家只要知道一个结论，就是取余运算没位运算快就行，所以使用了 与位运算

##### 怎么解决hash冲突的呢？

> 解决hash冲突的办法
>
> 1. 开放定址法（线性探测再散列，二次探测再散列，伪随机探测再散列）
>
>    > 核心思想：如果出现散列冲突，我们就**重新探测**一个空闲位置，再将元素插入。
>
> 2. 再哈希法
>
> 3. **链地址法**
>
> 4. 建立一个公共溢出区
>
> Java采用的是 链地址法 ，链地址法就是将相同hash值的对象组织成一个链表放在hash值对应的槽位，
>
> 在**插入**的时候，我们可以通过散列函数计算出对应的散列槽位，将元素插入到对应的链表即可，时间复杂度为**O（1）**；在**查找或删除**元素时，我们同样通过散列函数计算出对应的散列槽位，然后再通过遍历链表进行查找或删除，时间复杂度为**O（k）**，k为链表长度。

##### jdk1.7和jdk1.8在链表插入元素有什么区别嘛？

jdk1.7: 头插法，但是可能会出现无限循环问题

jdk.18:尾插法

头插法可能会出现死循环问题详解：

```java
void transfer(Entry[] newTable, boolean rehash) {
        int newCapacity = newTable.length;
        for (Entry<K,V> e : table) {
            while(null != e) {
            	//1， 获取旧表的下一个元素
                Entry<K,V> next = e.next;
                if (rehash) {
                    e.hash = null == e.key ? 0 : hash(e.key);
                }
                int i = indexFor(e.hash, newCapacity);
                e.next = newTable[i];
                newTable[i] = e;
                e = next;
            }
        }
    }
```

1.  假设旧表的初始长度为2，此时已经在下标为1的位置存放了两个元素，再put第三个元素的时候考虑需要扩容；

   ![](http://image.tinx.top/20211116100103.png)

2. 此刻有两个线程A,B都进行put操作，线程A先扩容，执行到代码Entry<K,V> next = e.next;执行完这段代码，线程A挂起；
   然后线程B开始执行transfer函数中的while循环，会把原来的table变成一个table（线程B自己的栈中），再写入到内存中。

   注意，因为线程A的e指向了key(3), next指向了key(7), 其在线程B rehash后，指向了线程B重组后的链表。我们可以看到链表的顺序被反转了。

   ![](http://image.tinx.top/20211116100125.png)

3. 线程A被唤醒，继续执行：

   先是执行newTable[i] = e ;
   然后是e = next , 导致了e指向了key(7);
   而下一次循环的next = e.next 导致next指向了key(3)
   如下图：

   ![](http://image.tinx.top/20211116100151.png)

4.  当前循环：
   e.next = newTable[i];
   newTable[i] = e ;
   e = next;
   将key(7)摘下来采用头插法，放到newTable[i]的第一个元素中，下一个结点指向key(3)
   下一次循环：
   next = e.next; 此时e为key(3)
   此时next = null; 不会在往下循环了。

   ![](http://image.tinx.top/20211116100213.png)

5. 此时key(3)采用头插法又放到newTable[i]的位置，导致key(3)指向key(7)，注意此时key(7).next已经指向了key(3)，所以环形链表就出现了。如下图：于是当我们的线程A调用get()方法时，如果下标映射到3处，则会出现死循环。

   ![](http://image.tinx.top/20211116100246.png)总结：
   线程A先执行，执行完Entry<K,V> next = e.next;这行代码后挂起，然后线程B完整的执行完整个扩容流程，接着线程A唤醒，继续之前的往下执行，当while循环执行3次后会形成环形链表。



#### Get流程

##### 源码

```java
// 代码面上意思的意思 (doge)
public V get(Object key) {
    Node<K,V> e;
    return (e = getNode(hash(key), key)) == null ? null : e.value;
}
```

```java
final Node<K,V> getNode(int hash, Object key) {
    Node<K,V>[] tab; Node<K,V> first, e; int n; K k;
  	// 判断当前Map对象是否为空，如果不为空，并且Map中数组的长度大于0，并且数组中的元素不为null
    if ((tab = table) != null && (n = tab.length) >0 && (first = tab[(n - 1) &hash]) != null) {
      // 查看第一个元素是否是要查找的值，如果是的话，就直接返回  
      if (first.hash == hash && ((k = first.key) == key || (key != null && key.equals(k))))
            return first;
      	// 如果 first链表的下一个元素不为Null
        if ((e = first.next) != null) {
          	// 如果首元素是 红黑树类型
            if (first instanceof TreeNode)
              	// 就从红黑树中查找这个元素
                return ((TreeNode<K,V>)first).getTreeNode(hash, key);
          	// 否则使用 do while 遍历这个是否和传入的key相同，会返回这个对象
            do {
                if (e.hash == hash && ((k = e.key) == key || (key != null && key.equals(k))))
                    return e;
            } while ((e = e.next) != null);
        }
    }
    return null;
}
```

#### HashMap 的长度为什么是2的幂次方

为了能让 HashMap 存取高效，尽量较少碰撞，也就是要尽量把数据分配均匀。我们上面也讲到了过了，Hash 值的范围值-2147483648到2147483647，前后加起来大概40亿的映射空间，只要哈希函数映射得比较均匀松散，一般应用是很难出现碰撞的。但问题是一个40亿长度的数组，内存是放不下的。所以这个散列值是不能直接拿来用的。用之前还要先做对数组的长度取模运算，得到的余数才能用来要存放的位置也就是对应的数组下标。这个数组下标的计算方法是“ `(n - 1) & hash`”。（n代表数组长度）。这也就解释了 HashMap 的长度为什么是2的幂次方。

这个算法应该如何设计呢？

我们首先可能会想到采用%取余的操作来实现。但是，重点来了：“取余(%)操作中如果除数是2的幂次则等价于与其除数减一的与(&)操作（也就是说 hash%length==hash&(length-1)的前提是 length 是2的 n 次方；）。” 并且 采用二进制位操作 &，相对于%能够提高运算效率，这就解释了 HashMap 的长度为什么是2的幂次方。

### LinkedHashMap

> 算是HashMap的升级版，内部使用的类Entry是继承自HashMap的Node类，向其中添加了两个字段 before、after
>
> ![](http://image.tinx.top/20211118095051.png)
>
> LinkedHashMap维护着一个运行于所有条目的双重链接列表。此链接列表定义了迭代顺序，该迭代顺序可以是插入顺序（insert-order）或者是访问顺序，其中默认的迭代访问顺序就是插入顺序，即可以按插入的顺序遍历元素
>
> - LRU算法可以用LinkedHashMap实现。

#### JAVA8的ConcurrentHashMap为什么放弃了分段锁，有什么问题吗，如果你来设计，你如何设计。
jdk8 放弃了分段锁而是用了Node锁，减低锁的粒度，提高性能，并使用CAS操作来确保Node的一些操作的原子性，取代了锁。

可以跟面试官聊聊悲观锁和CAS乐观锁的区别，优缺点哈~

### TreeMap

> 有序的Map集合

我们在创建时可以向构造函数中创建一个比较器 new Compartor() .... 用来比较集合中的元素，达到排序的作用



### HashTable

+ HashTable的键和值都不能为null，如果有Null则会抛出 NullPointerException
+ HashTable使用方式和HashMap是一致的，区别在于HashTable是线程安全的，HashMap是线程不安全的
+ 底层数组 HahTable$Entry[] 初始化大小为 11
+ 临界值为 11 * 0.75 = 8

#### 扩容机制

HahTable当遇到当前添加的值超过临界值时 11*0.75 = 8时，会选择将内部的容量 进行翻倍 + 1



### ConcurrentHashMap

> 初始数组容量为 16 增长因子为0.75（跟HashMap几乎相同），唯一的不同就是他进行了分块存储```Segement```(此类继承了ReentranLock，相当于在每一段都加了锁🔐)



## 总结

![](http://image.tinx.top/20211116154925.png)
