# [扁平化嵌套列表迭代器](https://leetcode-cn.com/problems/flatten-nested-list-iterator/)

给你一个嵌套的整型列表。请你设计一个迭代器，使其能够遍历这个整型列表中的所有整数。

列表中的每一项或者为一个整数，或者是另一个列表。其中列表的元素也可能是整数或是其他列表。

 

**示例 1:**

```
输入: [[1,1],2,[1,1]]
输出: [1,1,2,1,1]
解释: 通过重复调用 next 直到 hasNext 返回 false，next 返回的元素的顺序应该是: [1,1,2,1,1]。
```

**示例 2:**

```
输入: [1,[4,[6]]]
输出: [1,4,6]
解释: 通过重复调用 next 直到 hasNext 返回 false，next 返回的元素的顺序应该是: [1,4,6]。
```



本题的关键点就是 一个嵌套的整形列表，这个列表中每一项可能是一个数 或者又是一个嵌套列表，看到这里，```递归算法```还不整上等过年？

然后要有hasNext 和 next，我们可以使用数据结构 ```队列的容量和出队列```来解决，书写的代码一次性通过！特别开心！菜鸟终于有了提高~

![](http://image.tinx.top/img20210323103558.png)

答案:

```java
public class NestedIterator implements Iterator<Integer> {

    Queue<Integer> queue;

    public NestedIterator(List<NestedInteger> nestedList) {
        queue = new LinkedList<>();
        for (NestedInteger nestedInteger : nestedList) {
            getIndex(nestedInteger);
        }
    }

    /**
     * 现在最关键的就是假如说他的list里面的元素还是带着一个list，所以使用递归来解决
     *
     * 使用递归来返回一个数组
     */
    public void getIndex(NestedInteger res){
        if(!res.isInteger() && res.getList() == null) return;
        if(res.isInteger()) queue.add(res.getInteger());
        if(res.getList() != null){
            for (NestedInteger integer : res.getList()) {
                getIndex(integer);
            }
        }
    }

    @Override
    public boolean hasNext() {
      	// 直接判断队列的容量来解决是否还有元素的问题
        return queue.size() > 0;
    }

    @Override
    public Integer next() {
      	// 出队列来实现 next
        return queue.poll();
    }
}
```

