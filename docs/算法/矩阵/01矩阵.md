# [01 矩阵](https://leetcode-cn.com/problems/01-matrix/)

给定一个由 0 和 1 组成的矩阵，找出每个元素到最近的 0 的距离。

两个相邻元素间的距离为 1 。

**示例 1：**

```
输入：
[[0,0,0],
 [0,1,0],
 [0,0,0]]

输出：
[[0,0,0],
 [0,1,0],
 [0,0,0]]
```

**示例 2：**

```
输入：
[[0,0,0],
 [0,1,0],
 [1,1,1]]

输出：
[[0,0,0],
 [0,1,0],
 [1,2,1]]
```



## 暴力法 yyds

没啥好说的，直接两层for循环 + 递归法 搞定收工！（代码仅仅做展示用，因为面试一般不会让你用暴力法的，我不保证代码可行，仅仅做思路展示）

```java
// 暴力法 yyds
public static int[][] updateMatrix(int[][] mat) {
    int len = mat.length, wid = mat[0].length;
    int[][] res = new int[len][wid];
    for (int i = 0; i < len; i++) {
        for (int j = 0; j < wid; j++) {
            if(mat[i][j] != 0){
                // 如果目标二维数组不是0，说明需要找出他的邻居 / 他邻居的邻居是不是0，是0的话计算距离
                res[i][j] = getDistance(mat,0,i,j);
            }
            // 否则遍历的是0 说明距离0就是他本身 为0，不做任何操作
        }
    }
    return res;
}

// 递归的方式，全方位的找出上下左右的距离 仅仅作为思路来展示，具体代码不详细纠结
public static int getDistance(int[][] target,int distance,int x,int y){
    if(target[x][y] == 0){
        return distance;
    }

    int left = getDistance(target,++distance,x-1,y);
    int right = getDistance(target,++distance,x+1,y);
    int top = getDistance(target,++distance,x,y - 1);
    int bottom = getDistance(target,++distance,x,y + 1);
    // 找出最大的距离
    return Integer.max(Integer.max(Integer.max(left,right),top),bottom);
}
```



## BFS（广度优先搜索遍历）

注意哦，本题不是二叉树，但是可以套用二叉树的BFS，做过相关算法题的同志们，应该知道，BFS的实现原理就是使用空间换时间，一般使用队列、栈等数据结构存储临时值，然后对其进行数据处理，so..... here is answer~！（自己只会暴力法的思路，这是我看的答案 -_- 还是太菜了）

- 对于 **「图 的 BFS」 （「多源 BFS」）** 做法其实也是一样滴～，与 「Tree 的 BFS」的区别注意以下两条就 ok 辣～
  - Tree 只有 1 个 root，而图可以有多个源点，所以首先需要把多个源点都入队；
  - Tree 是有向的因此不需要标识是否访问过，而对于无向图来说，必须得标志是否访问过哦！并且为了防止某个节点多次入队，需要在其入队之前就将其设置成已访问！

本题的关键点：

就是想四周扩散，这就是 <strong>「图的广度优先搜索遍历」</strong>的精髓，很实用~！

```java
// 以此来依次扩散，求出相应的距离结果
int newX = x + dx[i];
int newY = y + dy[i];
// 判断当前的元素是否大小于0 防止数组越界Exception
if (newX >= 0 && newX < m && newY >= 0 && newY < n) {
    // 如果当前遍历结果是 -1 说明这个位置不是0，就要更新最新的距离 atrix[x][y] + 1
    if (matrix[newX][newY] == -1) {
        // 设置访问过的符号
        matrix[newX][newY] = matrix[x][y] + 1;
        // 然后重新加入队列 为后面的元素起到了累加的作用
        queue.offer(new int[]{newX, newY});
    }
}
```

答案：

```java
// BFS 广度优先搜索遍历
public static int[][] updateMatrix1(int[][] matrix) {
    // 首先将所有的 0 都入队，并且将 1 的位置设置成 -1，表示该位置是 未被访问过的 1
    Queue<int[]> queue = new LinkedList<>();
    int m = matrix.length, n = matrix[0].length;
    for (int i = 0; i < m; i++) {
        for (int j = 0; j < n; j++) {
            if (matrix[i][j] == 0) {
                queue.offer(new int[]{i, j});
            } else {
                matrix[i][j] = -1;
            }
        }
    }
    // x 轴的上下 应该为当前元素 上： x - 1 下： x + 1 故 -1 1 0 0
    // y 轴的左右 应该为当前元素 左： y -1 右：y+ 1 故 0 0 -1 1
    // dx dy 的作用仅仅是求出当前元素的上下左右的是否为0值
    int[] dx = new int[]{-1, 1, 0, 0};
    int[] dy = new int[]{0, 0, -1, 1};
    while (!queue.isEmpty()) {
        int[] point = queue.poll();
        int x = point[0], y = point[1];
        for (int i = 0; i < 4; i++) {
            // 这边 （dx, dy) 代表的依次从 0 - 3 分别为： 当前元素的 上、下、左、右
          	// 以此来依次扩散，求出相应的距离结果
            int newX = x + dx[i];
            int newY = y + dy[i];
            // 判断当前的元素是否大小于0 防止数组越界Exception
            if (newX >= 0 && newX < m && newY >= 0 && newY < n) {
                // 如果当前遍历结果是 -1 说明这个位置不是0，就要更新最新的距离 atrix[x][y] + 1
                if (matrix[newX][newY] == -1) {
                    // 设置访问过的符号
                    matrix[newX][newY] = matrix[x][y] + 1;
                    // 然后重新加入队列 为后面的元素起到了累加的作用
                    queue.offer(new int[]{newX, newY});
                }
            }
        }
    }
    return matrix;
}
```





## 动态规划

都看到这里了，就一定是一道动态规划的问题，那么还是那个老三样：

+ 初始状态

  ```int[][] res = new int[len+1][wid+1]```

+ 边界

+ 转移方程

```java
dp[i][j] = Math.min(dp[i][j], dp[i - 1][j] + 1);
```

答案：

```java
  public int[][] updateMatrix(int[][] matrix) {
    int m = matrix.length, n = matrix[0].length;
    int[][] dp = new int[m][n];
    for (int i = 0; i < m; i++) {
      for (int j = 0; j < n; j++) {
        dp[i][j] = matrix[i][j] == 0 ? 0 : -1;
      }
    }

    // 从左上角开始
    for (int i = 0; i < m; i++) {
      for (int j = 0; j < n; j++) {
        if (i - 1 >= 0) {
          dp[i][j] = Math.min(dp[i][j], dp[i - 1][j] + 1);
        }
        if (j - 1 >= 0) {
          dp[i][j] = Math.min(dp[i][j], dp[i][j - 1] + 1);
        }
      }
    }
    // 从右下角开始
    for (int i = m - 1; i >= 0; i--) {
      for (int j = n - 1; j >= 0; j--) {
        if (i + 1 < m) {
          dp[i][j] = Math.min(dp[i][j], dp[i + 1][j] + 1);
        }
        if (j + 1 < n) {
          dp[i][j] = Math.min(dp[i][j], dp[i][j + 1] + 1);
        }
      }
    }
    return dp;
  }
}
```

