# [比特位计数](https://leetcode-cn.com/problems/counting-bits/)

给定一个非负整数 **num**。对于 **0 ≤ i ≤ num** 范围中的每个数字 **i** ，计算其二进制数中的 1 的数目并将它们作为数组返回。

**示例 1:**

```
输入: 2
输出: [0,1,1]
```

**示例 2:**

```
输入: 5
输出: [0,1,1,2,1,2]
```

**进阶:**

- 给出时间复杂度为**O(n\*sizeof(integer))**的解答非常容易。但你可以在线性时间**O(n)**内用一趟扫描做到吗？
- 要求算法的空间复杂度为**O(n)**。
- 你能进一步完善解法吗？要求在C++或任何其他语言中不使用任何内置函数（如 C++ 中的 **__builtin_popcount**）来执行此操作。



其实本题的问题就是 求一个数 从0 -> 传入的 n 之间每一个元素的二进制1的个数



## 暴力解决办法

```java
public static int[] countBits(int n) {
    int[] dp = new int[n + 1];
    for (int i = 0; i <= n; i++) {
        dp[i] = countOne(i);
    }
    return dp;
}

public static int countOne(int num){
    int res = 0;
    while(num>0){
        num &= (num - 1);
        res++;
    }

    return res;
}
```



## 最高有效位

​	就是利用了 动态规划，然后根据之前的结果进行叠加的方式，本解决思路就是：

​	当前值和上一个值进行 ```与运算``` 如果结果为0,那么最高有效位就是当前遍历的数字

```java

public static int[] countBits1(int n) {
    int[] dp = new int[n + 1];
    int height = 0;
    for (int i = 1; i <= n; i++) {
        if ((i & (i - 1)) == 0) {
            height = i;
        }
        dp[i] = dp[i - height] + 1;
    }
    return dp;
}
```

