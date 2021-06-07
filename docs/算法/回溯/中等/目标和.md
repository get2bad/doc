# [目标和](https://leetcode-cn.com/problems/target-sum/)

给你一个整数数组 `nums` 和一个整数 `target` 。

向数组中的每个整数前添加 `'+'` 或 `'-'` ，然后串联起所有整数，可以构造一个 **表达式** ：

- 例如，`nums = [2, 1]` ，可以在 `2` 之前添加 `'+'` ，在 `1` 之前添加 `'-'` ，然后串联起来得到表达式 `"+2-1"` 。

返回可以通过上述方法构造的、运算结果等于 `target` 的不同 **表达式** 的数目。

 

**示例 1：**

```
输入：nums = [1,1,1,1,1], target = 3
输出：5
解释：一共有 5 种方法让最终目标和为 3 。
-1 + 1 + 1 + 1 + 1 = 3
+1 - 1 + 1 + 1 + 1 = 3
+1 + 1 - 1 + 1 + 1 = 3
+1 + 1 + 1 - 1 + 1 = 3
+1 + 1 + 1 + 1 - 1 = 3
```

**示例 2：**

```
输入：nums = [1], target = 1
输出：1
```



实在是尴尬，之前想的也是以下回溯算法不断试错，也就是变形版的递归，然后无法在一个方法中搞定，就no face的看了答案，然后我直呼我是个沙雕~ ╮(╯▽╰)╭



## 回溯算法(DFS)

所谓回溯算法，就是从0不断试错，最终如果满足这个条件，就将count数目 + 1，如果不满足的话，不会增加count值，很简单巧妙的解决方案

- 时间复杂度：O(2^n)，其中 n*n* 是数组nums的长度。回溯需要遍历所有不同的表达式，共有 2^n2*n* 种不同的表达式，每种表达式计算结果需要 O(1)*O*(1) 的时间，因此总时间复杂度是 O(2^n)*O*(2*n*)。
- 空间复杂度：O(n)，其中 n*n* 是数组 nums的长度。空间复杂度主要取决于递归调用的栈空间，栈的深度不超过 n*n*。

```java
int count = 0;

public int findTargetSumWays1(int[] nums, int target) {
    backTrack(nums,target,0,0);
    return count;
}

public void backTrack(int nums[], int target, int index, int sum) {
    if (index == nums.length) {
        if(target == sum){
            count ++;
        }
    } else {
        backTrack(nums,target,index + 1, sum + nums[index]);
        backTrack(nums,target,index + 1, sum - nums[index]);
    }
}
```



## 动态规划(DP)

// TODO

在学校里 莫名其妙的很烦躁！！！动态规划不会做，过几天回家好好整理下，先挖个坑吧

```java
public static int findTargetSumWays(int[] nums, int target) {
    int sum = 0;
    for (int i = 0; i < nums.length; i++) sum += nums[i];
    // 绝对值范围超过了sum的绝对值范围则无法得到
    if (Math.abs(target) > Math.abs(sum)) return 0;

    int len = nums.length;
    // - 0 +
    int t = sum * 2 + 1;
    int[][] dp = new int[len][t];
    // 初始化
    if (nums[0] == 0) {
        dp[0][sum] = 2;
    } else {
        dp[0][sum + nums[0]] = 1;
        dp[0][sum - nums[0]] = 1;
    }

    for (int i = 1; i < len; i++) {
        for (int j = 0; j < t; j++) {
            // 边界
            int l = (j - nums[i]) >= 0 ? j - nums[i] : 0;
            int r = (j + nums[i]) < t ? j + nums[i] : 0;
            dp[i][j] = dp[i - 1][l] + dp[i - 1][r];
        }
    }
    return dp[len - 1][sum + target];
}
```

