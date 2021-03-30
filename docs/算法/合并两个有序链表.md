## 11. 合并两个排序好的链表，合并后的链表也是保持顺序 -> MergeTwoSortedLists

稍微动一些脑筋题，不过也是简单题

```java
public class MergeTwoSortedLists {

    public static void main(String[] args) {
        ListNode a = new ListNode(1,new ListNode(3,null));
        ListNode b = new ListNode(2,new ListNode(4,null));
        System.out.println(mergeTwoLists(a, b));
    }

    public static ListNode mergeTwoLists(ListNode l1, ListNode l2) {
        // 声明一个空的 ListNode 对象，作为要返回的对象
        ListNode dummy = new ListNode(0);
        // 复制这个要返回的对象 (内存地址复制)
        ListNode head = dummy;
        // 循环方法传入的 ListNode，直到一方为null未知
        while(l1 != null && l2 != null){
            // 进行比较，如果l1.val < l2.val 那么就把head的 next 指向这个 l1.val
            if(l1.val < l2.val){
                head.next = l1;
                l1 = l1.next;
            } else {
                // 如果是 l1.val >= l2.val 那么就把head的 next 指向这个 l2.val
                head.next = l2;
                l2 = l2.next;
            }
            // 将头结点指向下一个，供下一次循环赋值
            head = head.next;
        }
        // 如果两个ListNode长度不相同，那么就进行直接追加(因为之前的while循环一定会循环到一个的next为null的情况)
        head.next = l1 == null? l2:l1;
        return dummy.next;
    }
}
```

