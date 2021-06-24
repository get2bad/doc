# [剑指 Offer 15. 二进制中1的个数](https://leetcode-cn.com/problems/er-jin-zhi-zhong-1de-ge-shu-lcof/)

请实现一个函数，输入一个整数（以二进制串形式），输出该数二进制表示中 1 的个数。例如，把 9 表示成二进制是 1001，有 2 位是 1。因此，如果输入 9，则该函数输出 2。

 

**示例 1：**

```
输入：00000000000000000000000000001011
输出：3
解释：输入的二进制串 00000000000000000000000000001011 中，共有三位为 '1'。
```

**示例 2：**

```
输入：00000000000000000000000010000000
输出：1
解释：输入的二进制串 00000000000000000000000010000000 中，共有一位为 '1'。
```

**示例 3：**

```
输入：11111111111111111111111111111101
输出：31
解释：输入的二进制串 11111111111111111111111111111101 中，共有 31 位为 '1'。
```



对于简单题，我直接重拳出击！！！！



## 不费脑子系列

### API

二选一，任君挑选

```java
// API 转换法
public static int hammingWeight1(int n) {
    return Integer.bitCount(n);
}
```

```java
public static int hammingWeight(int n) {
    int res = 0;
    for (char c : Integer.toBinaryString(n).toCharArray()) {
        if(c == '1'){
            res ++;
        }
    }
    return res;
}
```



## 位运算法

<img src="http://image.tinx.top/20210623184521.png" style="zoom:50%;" />

二选一

```java
public static int hammingWeight2(int n) {
    int res = 0;
    for (int i = 0; i < 32; i++) {
        res += (n>> i & 1) == 1? 1:0;
    }
    return res;
}
```

优化升级版

```java
public static int hammingWeight3(int n) {
    int ret = 0;
    while (n != 0) {
        n &= n - 1;
        ret++;
    }
    return ret;
}
```







