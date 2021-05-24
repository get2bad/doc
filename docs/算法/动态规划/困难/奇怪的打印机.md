# [奇怪的打印机](https://leetcode-cn.com/problems/strange-printer/)

有台奇怪的打印机有以下两个特殊要求：

- 打印机每次只能打印由 **同一个字符** 组成的序列。
- 每次可以在任意起始和结束位置打印新字符，并且会覆盖掉原来已有的字符。

给你一个字符串 `s` ，你的任务是计算这个打印机打印它需要的最少打印次数。

 

**示例 1：**

```
输入：s = "aaabbb"
输出：2
解释：首先打印 "aaa" 然后打印 "bbb"。
```

**示例 2：**

```
输入：s = "aba"
输出：2
解释：首先打印 "aaa" 然后在第二个位置打印 "b" 覆盖掉原来的字符 'a'。
```



通过以上的事实我们得到两点：

1. 我们知道当区间的两边字符相同时(`aba`), 它的打印次数可以从它的更小一层的子区间的打印次数而来
2. 当区间两边字符不同时(`abab`)，它的打印次数会取其子区间中的最优解，这一部分我们需要枚举所有的可能性
   ![](http://image.tinx.top/20210524110140.png)



本题，一看就可以使用动态规划来解决，既然又是动态规划问题，那么我们依旧要拿出那个老三样:

+ 初始数据结构

  ```java
  int[][] dp = new int[len][len];
  ```

+ 边界

+ 状态转移公式

![](http://image.tinx.top/20210524105145.png)



1. 打印一个字符串的次数为1，因此`dp[i][i] = 1`

2. 当字符串长度大于等于2时，判断是否两边区间字符相等

   ```
   s[i] == s[j]?
   ```

   - 如果相等，打印次数可以从子区间的打印次数转移而来`dp[i][j] = dp[i][j-1];`。例如`aba`的打印次数由`ab`的打印次数决定。
   - 如果不相等，则枚举所有的可能组合，然后取其最优解。

   以`abab`做一次示例：

   ![](http://image.tinx.top/20210524105426.png)

```java
public static int strangePrinter(String s) {
    int len = s.length();
    int[][] dp = new int[len][len];

    for (int i = len - 1; i >= 0; i--) {
        dp[i][i] = 1;
        for (int j = i + 1; j < len; j++) {
            if (s.charAt(i) == s.charAt(j)) {
                dp[i][j] = dp[i][j - 1];
            } else {
                int minn = Integer.MAX_VALUE;
                for (int k = i; k < j; k++) {
                    // k = i, i+ 1, i + 2, i+ 3 以此类推
                    // j = i + 1, i+ 2, i + 3, i + 4 一次类推
                    minn = Math.min(minn, dp[i][k] + dp[k + 1][j]);
                }
                dp[i][j] = minn;
            }
        }
    }

    return dp[0][len - 1];
}
```

