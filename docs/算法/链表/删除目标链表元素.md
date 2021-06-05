# [移除链表元素](https://leetcode-cn.com/problems/remove-linked-list-elements/)

给你一个链表的头节点 `head` 和一个整数 `val` ，请你删除链表中所有满足 `Node.val == val` 的节点，并返回 **新的头节点** 。

 

**示例 1：**

<img src="http://image.tinx.top/20210605093456.png" style="zoom:50%;" />

```
输入：head = [1,2,6,3,4,5,6], val = 6
输出：[1,2,3,4,5]
```

**示例 2：**

```
输入：head = [], val = 1
输出：[]
```

**示例 3：**

```
输入：head = [7,7,7,7], val = 7
输出：[]
```



## 遍历

这种链表题做多了，自然而然的就会直接想到遍历，声明一个临时的链表，使用while进行遍历，找到相同数的直接跳过，将这个指针引到下一个指针继续判断

```java
// 迭代
public ListNode removeElements1(ListNode head, int val) {
    ListNode dummyHead = new ListNode(0);
    dummyHead.next = head;
    ListNode temp = dummyHead;
    while (temp.next != null) {
        if (temp.next.val == val) {
            temp.next = temp.next.next;
        } else {
            temp = temp.next;
        }
    }
    return dummyHead.next;
}
```



## 递归

拿到这道题，递归Terminator(终结者)上线了，这种题还不得妥妥的递归来搞定？

对于简单题，直接递归重拳出击.

<img src="http://image.tinx.top/20210605093807.png" style="zoom:67%;" />

```java
// 递归
public ListNode removeElements(ListNode head, int val) {
    // 递归终结表达式
    if(head == null ){
        return head;
    }
    head.next = removeElements(head.next,val);
    if(head.val == val){
        return head.next;
    }
    return head;
}
```

