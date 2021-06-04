# [相交链表](https://leetcode-cn.com/problems/intersection-of-two-linked-lists/)

编写一个程序，找到两个单链表相交的起始节点。

如下面的两个链表**：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_statement.png)](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_statement.png)

在节点 c1 开始相交。

 

**示例 1：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_1.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_1.png)

```
输入：intersectVal = 8, listA = [4,1,8,4,5], listB = [5,0,1,8,4,5], skipA = 2, skipB = 3
输出：Reference of the node with value = 8
输入解释：相交节点的值为 8 （注意，如果两个链表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [4,1,8,4,5]，链表 B 为 [5,0,1,8,4,5]。在 A 中，相交节点前有 2 个节点；在 B 中，相交节点前有 3 个节点。
```

 

**示例 2：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_2.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_2.png)

```
输入：intersectVal = 2, listA = [0,9,1,2,4], listB = [3,2,4], skipA = 3, skipB = 1
输出：Reference of the node with value = 2
输入解释：相交节点的值为 2 （注意，如果两个链表相交则不能为 0）。从各自的表头开始算起，链表 A 为 [0,9,1,2,4]，链表 B 为 [3,2,4]。在 A 中，相交节点前有 3 个节点；在 B 中，相交节点前有 1 个节点。
```

 

**示例 3：**

[![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/12/14/160_example_3.png)](https://assets.leetcode.com/uploads/2018/12/13/160_example_3.png)

```
输入：intersectVal = 0, listA = [2,6,4], listB = [1,5], skipA = 3, skipB = 2
输出：null
输入解释：从各自的表头开始算起，链表 A 为 [2,6,4]，链表 B 为 [1,5]。由于这两个链表不相交，所以 intersectVal 必须为 0，而 skipA 和 skipB 可以是任意值。
解释：这两个链表不相交，因此返回 null。
```

**注意：**

- 如果两个链表没有交点，返回 `null`.
- 在返回结果后，两个链表仍须保持原有的结构。
- 可假定整个链表结构中没有循环。
- 程序尽量满足 O(*n*) 时间复杂度，且仅用 O(*1*) 内存。



本题，如果不考虑注意的问题，那么我们可以直接使用暴力枚举法，来解决问题

```java
/**
 * 肯定要遍历这两个链表：+
 * 暴力法
 * 但是达不到题目要求的 时间复杂度 O(N) 空间复杂度 O(1)
 * 如果两个链表有相同的值，或者两者的对象地址相等，那么就是开始相交
 * 否则就是没有相交
 */
public static ListNode getIntersectionNode(ListNode headA, ListNode headB) {
    ListNode source = headB;
    while(headA!=null){
        while(headB != null){
            if(headA == headB) return headA;
            headB = headB.next;
            if(headB == null) {
                headB = source;
                break;
            }
        }
        headA = headA.next;
    }
    return null;
}
```

但是题目的注意事项里面明确要求：

**时间复杂度：O(N),空间复杂度O(1)**，所以并不能解决

这个最困扰的就是时间复杂度O(N)



思来想去，看了看答案思路(没错，我又看答案了)，发现，竟然看不懂.....没办法，画图思考吧

```java
/**
 * 双指针法
 */
public static ListNode getIntersectionNode1(ListNode headA, ListNode headB) {
    /**
     定义两个指针, 第一轮让两个到达末尾的节点指向另一个链表的头部, 最后如果相遇则为交点(在第一轮移动中恰好抹除了长度差)
     两个指针等于移动了相同的距离, 有交点就返回, 无交点就是各走了两条指针的长度
     **/
    if(headA == null || headB == null) return null;
    ListNode pA = headA, pB = headB;
    // 在这里第一轮体现在pA和pB第一次到达尾部会移向另一链表的表头,
    // 而第二轮体现在如果pA或pB相交就返回交点, 不相交最后就是null==null
    while(pA != pB) {
        pA = pA == null ? headB : pA.next;
        pB = pB == null ? headA : pB.next;
    }
    return pA;
}
```

我们来思索一下流程吧：

因为是双指针法，所以一定要

1. 声明两个指针，分别指向传入的 ```headA```,```headB```的头指针

2. 使用```while```循环，终止条件是 pA = pB （如果没有相交的结束终点是 null = null）

3. 循环内的主要 思想就是 

   + 如果pA 遍历完了，就会重新切换到 headB，继续遍历
   + 如果pB 遍历完了，就会重新切换到headA, 继续遍历

4. **重点：**

   + pA走过的路径： pA + pB

   + pB走过的路径： pB + pA
   + 走完后，如果没有重叠 就跳出循环 返回 null

5. 小例子：

   > pA: 1 -> 2 -> 3 -> 4
   >
   > pB: 9 -> 5 -> 4
   >
   > 
   >
   > pA走的路径： 1 -> 2 -> 3 -> 4 -> null -> 9 -> 5 -> 4 -> null
   > pB走的路径： 9 -> 5 -> 4 -> null -> 1 -> 2 -> 3 -> 4 -> null
   >
   > 
   >
   > 最终在null之前找到了两者相交的点 **4**，这就是关键点！



分析完了，这道题就很容易解决了。所以自己的算法还是要加强训练啊，哎，太菜了！！