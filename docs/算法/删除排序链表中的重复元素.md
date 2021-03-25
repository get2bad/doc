## 22. 删除排序链表中的重复元素

> 给定一个排序链表，删除所有重复的元素，使得每个元素只出现一次。
>
> 示例 1:
>
> 输入: 1->1->2
> 输出: 1->2
> 示例 2:
>
> 输入: 1->1->2->3->3
> 输出: 1->2->3

解题方案：

1.使用递归的方案

	+ 进行嵌套递归，如果当前递归的值和下一个相等，那么就将当前递归的下一个值赋值给当前的递归值

```java
public static ListNode deleteDuplicates(ListNode head) {
    if(head == null || head.next == null){
        return head;
    }
    head.next = deleteDuplicates(head.next);
    if(head.val == head.next.val) head = head.next;
    return head;
}
```

2.使用遍历的方案

	+ 创建一个新的链表，内存地址是head的内存地址(两个是同一份) 然后进行遍历这个新的链表

```java
public static ListNode deleteDuplicates1(ListNode head) {
    ListNode tmp = head;
    while(tmp != null && tmp.next != null){
        if(tmp.val == tmp.next.val){
            // 为什么不是 tmp = tmp.next.next? 因为这样什么都没有改变
            tmp.next = tmp.next.next;
        } else {
            tmp = tmp.next;
        }
    }
    return head;
}
```



## 82.[删除排序链表中的重复元素 II](https://leetcode-cn.com/problems/remove-duplicates-from-sorted-list-ii/)

存在一个按升序排列的链表，给你这个链表的头节点 `head` ，请你删除链表中所有存在数字重复情况的节点，只保留原始链表中 **没有重复出现** 的数字。

返回同样按升序排列的结果链表。

 

**示例 1：**

![img](https://assets.leetcode.com/uploads/2021/01/04/linkedlist1.jpg)

```
输入：head = [1,2,3,3,4,4,5]
输出：[1,2,5]
```

**示例 2：**

![img](https://assets.leetcode.com/uploads/2021/01/04/linkedlist2.jpg)

```
输入：head = [1,1,1,2,3]
输出：[2,3]
```



本题的关键点就是跳过那些重复的元素，以下是解题思路：

1. 遍历这个链表
   1. 如果当前遍历的下一个不为空，或者 当前遍历节点 不等于 下一个结点的值(相当于查看当前结点不等于下一个结点的值，说明这就是独一无二的)就将尾结点赋值为当前的遍历结点
   2. 然后使用一个循环重新遍历当前结点以下的结点，目的是过滤完那种不和当前结点相等的结点
   3. head = head.next; 继续往下遍历
2. 最后截断尾结点的下一个循环
3. 返回

```java
public static ListNode deleteDuplicates(ListNode head) {
    ListNode res = new ListNode( -1);
    ListNode tmp = res;
    while(head != null){
        if(head.next == null || head.val != head.next.val){
            // 跳过相同的节点
            tmp.next = head;
            // 然后将 尾结点替换到 tmp.next
            tmp = tmp.next;
        }
        // 如果当前值 和下一个值相等，就持续向下遍历
        while(head.next !=null && head.val == head.next.val){
            head = head.next;
        }
        // 往下继续遍历
        head = head.next;
    }
    // 截断后面的尾部
    tmp.next = null;

    return res.next;
}
```