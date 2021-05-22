# [最小路径和](https://leetcode-cn.com/problems/minimum-path-sum/)

给定一个包含非负整数的 `m x n` 网格 `grid` ，请找出一条从左上角到右下角的路径，使得路径上的数字总和为最小。

**说明：**每次只能向下或者向右移动一步。

 

**示例 1：**

![img](https://assets.leetcode.com/uploads/2020/11/05/minpath.jpg)

```
输入：grid = [[1,3,1],[1,5,1],[4,2,1]]
输出：7
解释：因为路径 1→3→1→1→1 的总和最小。
```

**示例 2：**

```
输入：grid = [[1,2,3],[4,5,6]]
输出：12
```



动态规划问题：

所谓动态规划的核心问题就是：

1. 空间换时间，多使用二维数组来解决问题

   ```int[][] source = new int[len][wid];```

2. 找到初始状态

   ```java
   source[0][0] = grid[0][0];
   for (int i = 1; i < len; i++) {
       source[i][0] = source[i-1][0] + grid[i][0];
   }
   for (int i = 1; i < wid; i++) {
       source[0][i] = source[0][i -1] + grid[0][i];
   }
   ```

3. 找到状态转移公式

   ```
   int first = source[i -1][j] + grid[i][j];
   int second = source[i][j - 1] + grid[i][j];
   source[i][j] = Math.min(first,second);
   ```

所以答案为：

> 由于路径的方向只能是向下或向右，因此网格的第一行的每个元素只能从左上角元素开始向右移动到达，网格的第一列的每个元素只能从左上角元素开始向下移动到达，此时的路径是唯一的，因此每个元素对应的最小路径和即为对应的路径上的数字总和。
>
> 对于不在第一行和第一列的元素，可以从其上方相邻元素向下移动一步到达，或者从其左方相邻元素向右移动一步到达，元素对应的最小路径和等于其上方相邻元素与其左方相邻元素两者对应的最小路径和中的最小值加上当前元素的值。由于每个元素对应的最小路径和与其相邻元素对应的最小路径和有关，因此可以使用动态规划求解。
>
> 创建二维数组source，与原始网格的大小相同，source`[i][j]`表示从左上角出发到 (i,j)(*i*,*j*) 位置的最小路径和。显然，source`[0][0]`=*grid*`[0][0]`。对于 dp 中的其余元素，通过以下状态转移方程计算元素值。
>
> - 当 i>0且 j=0时，source`[i][0]`=*source*`[i−1][0]`+*grid*`[i][0]`。
> - 当 i=0 且 j>0 时，source`[0][j]`=source`[0][j−1]`+*grid*`[0][j]`。
> - 当 i>0 且 j>0 时，*source*`[i][j]`=min(source`[i−1][j]`,source`[i][j−1]`)+*grid*`[i][j]`。
>
> 最后得到source`[i-1][j-1]`的值即为从网格左上角到网格右下角的最小路径和。

```java
public static int minPathSum(int[][] grid) {
    int len = grid.length;
    int wid = grid[0].length;
    int[][] source = new int[len][wid];
    source[0][0] = grid[0][0];
    for (int i = 1; i < len; i++) {
        source[i][0] = source[i-1][0] + grid[i][0];
    }
    for (int i = 1; i < wid; i++) {
        source[0][i] = source[0][i -1] + grid[0][i];
    }
    for (int i = 1; i < len; i++) {
        for (int j = 1; j < wid; j++) {
            int first = source[i -1][j] + grid[i][j];
            int second = source[i][j - 1] + grid[i][j];
            source[i][j] = Math.min(first,second);
        }
    }
    return source[len - 1][wid - 1];
}
```



总结：

本题算是前几天刷的那个**不同路径**问题的变种，他的本质就是决策两个不同的值的大小问题，如果小就进行赋值，我感觉本题应该归类为 <font color=green>简单</font>问题