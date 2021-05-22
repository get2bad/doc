# [不相交的线](https://leetcode-cn.com/problems/uncrossed-lines/)

在两条独立的水平线上按给定的顺序写下 `nums1` 和 `nums2` 中的整数。

现在，可以绘制一些连接两个数字 `nums1[i]` 和 `nums2[j]` 的直线，这些直线需要同时满足满足：

-  `nums1[i] == nums2[j]`
- 且绘制的直线不与任何其他连线（非水平线）相交。

请注意，连线即使在端点也不能相交：每个数字只能属于一条连线。

以这种方法绘制线条，并返回可以绘制的最大连线数。

 

**示例 1：**

<img src="https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/04/28/142.png" alt="img" style="zoom:8%;" />

```
输入：nums1 = [1,4,2], nums2 = [1,2,4]
输出：2
解释：可以画出两条不交叉的线，如上图所示。 
但无法画出第三条不相交的直线，因为从 nums1[1]=4 到 nums2[2]=4 的直线将与从 nums1[2]=2 到 nums2[1]=2 的直线相交。
```

**示例 2：**

```
输入：nums1 = [2,5,1,2,5], nums2 = [10,5,2,1,5,2]
输出：3
```

**示例 3：**

```
输入：nums1 = [1,3,7,1,7,5], nums2 = [1,9,2,5,1]
输出：2
```

 

这道题看完示例后，发现这个是一个动态规划问题，查找最大公共子序列的问题

既然是 动态规划问题，那么离不开两个点：

1. 初始状态，使用合理的数据结构来解决本题

   > 最终选择采用 二维数组的方式:
   >
   > int\[\]\[\] dp = new int\[len+1]\[wid+1]

2. 终止边界，找到状态转移方程

![](http://image.tinx.top/20210521102727.png)



最终流程(摘抄自[官方答案](https://leetcode-cn.com/problems/uncrossed-lines/solution/bu-xiang-jiao-de-xian-by-leetcode-soluti-6tqz/))

![](https://assets.leetcode-cn.com/solution-static/1035/1.png)

答案显而易见，可以使用双层for循环的方式，将这个dp二维数组填满，如果遇到了相等的情况，就对协上角的答案+1，最终输出dp二维数组的最后一个答案即可。

```java
public static int maxUncrossedLines(int[] nums1, int[] nums2) {
    int len = nums1.length, wid = nums2.length;
    int[][] dp = new int[len + 1][wid + 1];
    for (int i = 1; i <= len; i++) {
        int num1 = nums1[i - 1];
        for (int j = 1; j <= wid; j++) {
            int num2 = nums2[j - 1];
            if (num1 == num2) {
                dp[i][j] = dp[i - 1][j - 1] + 1;
            } else {
                dp[i][j] = Math.max(dp[i - 1][j], dp[i][j - 1]);
            }
        }
    }
    return dp[len][wid];
}
```



