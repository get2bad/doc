
# [一和零](https://leetcode-cn.com/problems/ones-and-zeroes/)

给你一个二进制字符串数组 `strs` 和两个整数 `m` 和 `n` 。

请你找出并返回 `strs` 的最大子集的大小，该子集中 **最多** 有 `m` 个 `0` 和 `n` 个 `1` 。

如果 `x` 的所有元素也是 `y` 的元素，集合 `x` 是集合 `y` 的 **子集** 。

 

**示例 1：**

```
输入：strs = ["10", "0001", "111001", "1", "0"], m = 5, n = 3
输出：4
解释：最多有 5 个 0 和 3 个 1 的最大子集是 {"10","0001","1","0"} ，因此答案是 4 。
其他满足题意但较小的子集包括 {"0001","1"} 和 {"10","1","0"} 。{"111001"} 不满足题意，因为它含 4 个 1 ，大于 n 的值 3 。
```

**示例 2：**

```
输入：strs = ["10", "0", "1"], m = 1, n = 1
输出：2
解释：最大的子集是 {"0", "1"} ，所以答案是 2 。
```

 

经过仔细读题，发现题目的要求仅仅是：

给你一个数组，然后让你输出满足传入的m个0 ，n个1的结果，

所以第一个想法就是使用暴力法：

## 暴力法

所谓暴力法，就是使用暴力迭代的方法去查找。

那如何去判断0 和1的大小呢？拆开？不存在的，直接进行Integer.parseInt(XXX)，然后进行与或判断，找出 0 和 1的个数，如果满足就 + 1。

然后对这个 0 数组 和1 数组 随机组合，使用前缀和的方式来查看是否满足要求，满足要求，就返回其结果, 由于不是最优解，就仅仅做到找出 0 和 1 的总数就停止了，后期待补充

```java
public static int findMaxForm(String[] strs, int m, int n) {
    int len = strs.length;
    int[]  oneSource= new int[len];
    int[]  zeroSource = new int[len];
    // 循环这个
    for (int i = 0; i < len; i++) {
        int element = Integer.parseInt(strs[i]), eleCnt = strs[i].length();

        int oneCnt = 0, zeroCnt = 0;
        // 1的个数
        //假设是1010，第一次从0开始不动，1右移变成0101,与运算成立，count+1，
        // 第二次，0010，不符合，第三次，0001，符合，count++
        for (int j = 0; j < eleCnt; j++) {
            if (((element >> j) & 1) == 1) {
                oneCnt++;
            }
        }
        // 0的个数
        zeroCnt += eleCnt - oneCnt;

        oneSource[i] = oneCnt;
        zeroSource[i] = zeroCnt;
    }
    return len;
}
```

### 错误解题

由于题目没有带连续两个字，所以本答案<delete>作废</delete>

```java
public int findMaxForm(String[] strs, int m, int n) {
    int len = strs.length;
    int oneCnt = 0, zeroCnt = 0;
    List<Integer> res = new LinkedList<>();
    // 循环这个
    for (int i = 0; i < len; i++) {
        int element = Integer.parseInt(strs[i]);
        // 1的个数
        for (int j = 0; j < 32; j++) {  
            if (((element >> j) & 1) == 1) {
                oneCnt++;
            }
        }
        // 0的个数
        zeroCnt += strs[i].length() - oneCnt;

        if (m == zeroCnt && n == oneCnt) {
            oneCnt = 0;
            zeroCnt = 0;
        }
        // 将目标数字加入数据集中
        res.add(element);

        if (zeroCnt > m || oneCnt > n) {
            oneCnt = 0;
            zeroCnt = 0;
            // 清除这里面的元素
            res.clear();
        }
    }
    return res.size();
}
```



## 动态规划法

动态规划法，老三样：

1. 定义状态

   ```dp[i][j][k]```

   > `dp[i][j][k]` 表示输入字符串在子区间 `[0, i]` 能够使用 `j` 个 `0` 和 `k` 个 `1` 的字符串的最大数量。

2. 状态转移方程

   ![](http://image.tinx.top/20210606092226.png)

3. 边界

输出状态是：

```dp[len][m][n]```

思路：把总共的 `0` 和 `1` 的个数视为背包的容量，每一个字符串视为装进背包的物品。这道题就可以使用 0-1 背包问题的思路完成，这里的目标值是能放进背包的字符串的数量。



<h2>注意：这个状态转移公式，是动态规划的重中之重</h2>

dp的最终结果(以题目示例1举例)：

```text
1 = {int[4]@494} [0, 0, 0, 0]
2 = {int[4]@495} [0, 0, 0, 0]
3 = {int[4]@496} [0, 0, 0, 0]
4 = {int[4]@497} [0, 0, 0, 0]
5 = {int[4]@498} [0, 0, 0, 0]
```

```text
0 = {int[4]@499} [0, 0, 0, 0]
1 = {int[4]@500} [0, 1, 1, 1]
2 = {int[4]@501} [0, 1, 1, 1]
3 = {int[4]@502} [0, 1, 1, 1]
4 = {int[4]@503} [0, 1, 1, 1]
5 = {int[4]@504} [0, 1, 1, 1]
```

```text
0 = {int[4]@505} [0, 0, 0, 0]
1 = {int[4]@506} [0, 1, 1, 1]
2 = {int[4]@507} [0, 1, 1, 1]
3 = {int[4]@508} [0, 1, 1, 1]
4 = {int[4]@509} [0, 1, 2, 2]
5 = {int[4]@510} [0, 1, 2, 2]
```

```text
0 = {int[4]@511} [0, 0, 0, 0]
1 = {int[4]@512} [0, 1, 1, 1]
2 = {int[4]@513} [0, 1, 1, 1]
3 = {int[4]@514} [0, 1, 1, 1]
4 = {int[4]@515} [0, 1, 2, 2]
5 = {int[4]@516} [0, 1, 2, 2]
```

```text
0 = {int[4]@517} [0, 1, 1, 1]
1 = {int[4]@518} [0, 1, 2, 2]
2 = {int[4]@519} [0, 1, 2, 2]
3 = {int[4]@520} [0, 1, 2, 2]
4 = {int[4]@521} [0, 1, 2, 3]
5 = {int[4]@522} [0, 1, 2, 3]
```

```text
0 = {int[4]@523} [0, 1, 1, 1]
1 = {int[4]@524} [1, 2, 2, 2]
2 = {int[4]@525} [1, 2, 3, 3]
3 = {int[4]@526} [1, 2, 3, 3]
4 = {int[4]@527} [1, 2, 3, 3]
5 = {int[4]@487} [1, 2, 3, 4]
```

最终返回的是 最后一项的 5 = {int[4]@487} [1, 2, 3, 4] 最后一项 4，是最终结果

<img src="http://image.tinx.top/20210606094043.png" style="zoom:50%;" />

```java
public int findMaxForm(String[] strs, int m, int n) {
    int len = strs.length;
    int[][][] dp = new int[len + 1][m + 1][n + 1];

    for (int i = 1; i <= len; i++) {
        // 注意：有一位偏移
        int[] count = countZeroAndOne(strs[i - 1]);
        for (int j = 0; j <= m; j++) {
            for (int k = 0; k <= n; k++) {
                // 先把上一行抄下来
                dp[i][j][k] = dp[i - 1][j][k];
                int zeros = count[0];
                int ones = count[1];
                if (j >= zeros && k >= ones) {
                    dp[i][j][k] = Math.max(dp[i - 1][j][k], dp[i - 1][j - zeros][k - ones] + 1);
                }
            }
        }
    }
    return dp[len][m][n];
}

private int[] countZeroAndOne(String str) {
    int[] cnt = new int[2];
    for (char c : str.toCharArray()) {
        cnt[c - '0']++;
    }
    return cnt;
}
```

