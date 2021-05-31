# [4的幂](https://leetcode-cn.com/problems/power-of-four/)

给定一个整数，写一个函数来判断它是否是 4 的幂次方。如果是，返回 `true` ；否则，返回 `false` 。

整数 `n` 是 4 的幂次方需满足：存在整数 `x` 使得 `n == 4x`

 

**示例 1：**

```
输入：n = 16
输出：true
```

**示例 2：**

```
输入：n = 5
输出：false
```

**示例 3：**

```
输入：n = 1
输出：true
```



刚看到这个题，切，简单题，哥哥我直接while循环就完事了，然后就碰到了 超时 + 超限问题，哦豁，完蛋

```java
public static boolean isPowerOfFour(int n) {
    int dp = 0,source = 0;
    while(source <= n){
        source = (int)Math.pow(4,dp);
        if(source == n){
            return true;
        }
        dp++;
    }
    return false;
}
```

## 位运算解法1

然后，思考到，我们其实可以使用位运算来解决呀，因为4的次幂 = 2的2次幂，所以呢，这个就是简单的二进制位运算的问题了，那么答案手到擒来~

```java
public static boolean isPowerOfFour1(int n) {
    if (n <= 0) return false;
    // 先对这个值进行开根号
    int x = (int)Math.sqrt(n);
    // 如果 开根后的结果不等于传入的说明不是2的次幂，直接返回 false
    // 开根后的结果 和 其 负值 进行与运算 如果不相等 原值就返回 false
    // 如果是9 这种 和 自己的负数开根 3 & -3 那么结果就是1 和之前的不同，说明不是2的次幂
    return x * x == n && (x & -x) == x;
}
```

## 位运算解法2

1. 和昨天的每日一题类似，先判断 n 是否大于 0 ，是则转向第二步，否则返回false。
2. n 如果是 4 的幂，那肯定是 2 的幂，所以先求解**n & (n - 1)**（删掉 n 的二进制表示最右边的 1 ），判断是否为 0 ，不为 0 ，返回false， **否则 n 是 2 的幂**，调到第三步。
3. 这个时候只需判断 n 最右边的 1 是否在奇数位置，通过 n 向右移位操作，用 cnt 记数，判断 cnt 的奇偶性既可。

```java
class Solution {
    public boolean isPowerOfFour(int n) {
        if(n <= 0) return false;
        if((n & (n - 1)) != 0) return false;
        int cnt = 0;
        while(n != 0) {
            n >>= 1;
            cnt++;
        }
        return cnt % 2 == 1;
    }
}
```

