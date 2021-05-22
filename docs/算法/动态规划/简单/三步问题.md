# [试题 08.01. 三步问题](https://leetcode-cn.com/problems/three-steps-problem-lcci/)

三步问题。有个小孩正在上楼梯，楼梯有n阶台阶，小孩一次可以上1阶、2阶或3阶。实现一种方法，计算小孩有多少种上楼梯的方式。结果可能很大，你需要对结果模1000000007。

示例1:

 输入：n = 3 
 输出：4
 说明: 有四种走法

示例2:

 输入：n = 5
 输出：13



本题是一个动态规划的问题，那么就是老三样

+ 初始状态

  > Int\[] dp = new int\[n+1];

+ 寻找边界

  > 边界就是n节台阶

+ 状态转移公式

  > dp[i] = ((dp[i - 1] + dp[i - 2]) % 1000000007 + dp[i - 3]) % 1000000007

  其实说到底就是找规律，从第四项的结果等前三项的和 dp\[3] = dp\[0] + dp\[1] + dp[2]

```java
public static int waysToStep(int n) {
    if (n == 1 || n == 2) {
        return n;
    }
    if (n == 3) {
        return 4;
    }
    int[] dp = new int[n + 1];
    dp[1] = 1;
    dp[2] = 2;
    dp[3] = 4;
    //取模运算,得分步取
    for (int i = 4; i <= n; i++) {
        dp[i] = ((dp[i - 1] + dp[i - 2]) % 1000000007 + dp[i - 3]) % 1000000007;
    }
    return dp[n];
}
```

