# [不同路径](https://leetcode-cn.com/problems/unique-paths/)

一个机器人位于一个 m x n 网格的左上角 （起始点在下图中标记为 “Start” ）。

机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为 “Finish” ）。

问总共有多少条不同的路径？

![](https://assets.leetcode.com/uploads/2018/10/22/robot_maze.png) 

示例 1：

输入：m = 3, n = 7
输出：28

示例 2：

输入：m = 3, n = 2
输出：3
解释：
从左上角开始，总共有 3 条路径可以到达右下角。

1. 向右 -> 向下 -> 向下
2. 向下 -> 向下 -> 向右
3. 向下 -> 向右 -> 向下



示例 3：

输入：m = 7, n = 3
输出：28

示例 4：输入：m = 3, n = 3
输出：6



因为本题是一个 动态规划的问题，所以我们先来思考相关的条件：

我们令 dp`[i][j]` 是到达 i, j 最多路径

动态方程：dp`[i][j]` = dp`[i-1][j]` + dp`[i][j-1]`

注意，对于第一行 dp`[0][j]`，或者第一列 dp`[i][0]`，由于都是在边界，所以只能为 1

时间复杂度：O(m*n)O(m∗n)

空间复杂度：O(m * n)O(m∗n)

优化：因为我们每次只需要 dp`[i-1][j]`,dp`[i][j - 1]`

```
public static int uniquePaths(int m, int n) {
    int[][] dp = new int[m][n];
    for (int i = 0; i < n; i++) dp[0][i] = 1;
    for (int i = 0; i < m; i++) dp[i][0] = 1;
    for (int i = 1; i < m; i++) {
        for (int j = 1; j < n; j++) {
            dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
        }
    }
    return dp[m - 1][n - 1];
}

// 优化版
public static int uniquePaths1(int m, int n) {
    int[] cur = new int[n];
    Arrays.fill(cur,1);
    for (int i = 1; i < m;i++){
        for (int j = 1; j < n; j++){
            cur[j] += cur[j-1] ;
        }
    }
    return cur[n-1];
}
```