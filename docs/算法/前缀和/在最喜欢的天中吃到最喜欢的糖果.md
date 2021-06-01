# [你能在你最喜欢的那天吃到你最喜欢的糖果吗？](https://leetcode-cn.com/problems/can-you-eat-your-favorite-candy-on-your-favorite-day/)

给你一个下标从 **0** 开始的正整数数组 `candiesCount` ，其中 `candiesCount[i]` 表示你拥有的第 `i` 类糖果的数目。同时给你一个二维数组 `queries` ，其中 `queries[i] = [favoriteTypei, favoriteDayi, dailyCapi]` 。

你按照如下规则进行一场游戏：

- 你从第 `**0**` 天开始吃糖果。
- 你在吃完 **所有** 第 `i - 1` 类糖果之前，**不能** 吃任何一颗第 `i` 类糖果。
- 在吃完所有糖果之前，你必须每天 **至少** 吃 **一颗** 糖果。

请你构建一个布尔型数组 `answer` ，满足 `answer.length == queries.length` 。`answer[i]` 为 `true` 的条件是：在每天吃 **不超过** `dailyCapi` 颗糖果的前提下，你可以在第 `favoriteDayi` 天吃到第 `favoriteTypei` 类糖果；否则 `answer[i]` 为 `false` 。注意，只要满足上面 3 条规则中的第二条规则，你就可以在同一天吃不同类型的糖果。

请你返回得到的数组 `answer` 。

 

**示例 1：**

```
输入：candiesCount = [7,4,5,3,8], queries = [[0,2,2],[4,2,4],[2,13,1000000000]]
输出：[true,false,true]
提示：
1- 在第 0 天吃 2 颗糖果(类型 0），第 1 天吃 2 颗糖果（类型 0），第 2 天你可以吃到类型 0 的糖果。
2- 每天你最多吃 4 颗糖果。即使第 0 天吃 4 颗糖果（类型 0），第 1 天吃 4 颗糖果（类型 0 和类型 1），你也没办法在第 2 天吃到类型 4 的糖果。换言之，你没法在每天吃 4 颗糖果的限制下在第 2 天吃到第 4 类糖果。
3- 如果你每天吃 1 颗糖果，你可以在第 13 天吃到类型 2 的糖果。
```

**示例 2：**

```
输入：candiesCount = [5,2,6,4,1], queries = [[3,1,2],[4,10,3],[3,10,100],[4,100,30],[1,3,1]]
输出：[false,true,true,false,false]
```



读了好久的题目，才理解这个意思，其中这个queries数组的的每个数组元素数组代表的意思是：

例如[3,1,2] => [最想吃的是第4类(3+1)类糖果，最想在第2(1+1)天吃到，每天最多吃2颗糖果]

然后输出的answer数组就是 上述的数组能否满足在最喜欢的那天能否吃到第X类糖果



大致步骤就是：

1. 先求出每一天的前缀和
2. 然后遍历这个quires数组，找出要在最喜欢的那天要吃的糖果的最大 / 最小值
3. 如果处在这个最大最小值之间，那么就可以达成目标 将当前置为true，否则 false

```java
public static boolean[] canEat(int[] candiesCount, int[][] queries) {
    int n  = candiesCount.length;
    boolean[] res = new boolean[queries.length];
    long[] sum = new long [n + 1];
    sum[0] = 0L;
    // 利用前缀和保存第 i 类糖果个数的区间
    for (int i = 1 ; i <= n ; i++) {
        sum[i] = sum[i - 1] + candiesCount[i - 1];
    }
    // 遍历这个需求数组，进行相关的判断处理
    for (int i = 0; i < queries.length; i ++) {
        // queries[i] = [favoriteTypei, favoriteDayi, dailyCapi]
        int favoriteType = queries[i][0];
        int favoriteDay = queries[i][1] + 1;
        int dailyCap = queries[i][2];
        // 在favoriteDayi日可以吃到的糖果个数区间 (第0天开始吃糖果)
        // 这里记得转类型，不然会溢出
        long total = (long) favoriteDay * dailyCap;
        // 最喜欢吃的糖果的个数的区间 最少 和 最多
        long favoriteMin = sum[favoriteType] + 1;
        long favoriteMax = sum[favoriteType + 1];
        // 判断两个区间是否存在交集，存在则在favoriteDay日可以吃到favoriteType糖果
        res[i] = favoriteMin <= total && favoriteDay <= favoriteMax;
    }
    return res;
}
```

