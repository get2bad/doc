给你一个二维矩阵 `matrix` 和一个整数 `k` ，矩阵大小为 `m x n` 由非负整数组成。

矩阵中坐标 `(a, b)` 的 **值** 可由对所有满足 `0 <= i <= a < m` 且 `0 <= j <= b < n` 的元素 `matrix[i][j]`（**下标从 0 开始计数**）执行异或运算得到。

请你找出 `matrix` 的所有坐标中第 `k` 大的值（**`k` 的值从 1 开始计数**）。

 

**示例 1：**

```
输入：matrix = [[5,2],[1,6]], k = 1
输出：7
解释：坐标 (0,1) 的值是 5 XOR 2 = 7 ，为最大的值。
```

**示例 2：**

```
输入：matrix = [[5,2],[1,6]], k = 2
输出：5
解释：坐标 (0,0) 的值是 5 = 5 ，为第 2 大的值。
```

**示例 3：**

```
输入：matrix = [[5,2],[1,6]], k = 3
输出：4
解释：坐标 (1,0) 的值是 5 XOR 1 = 4 ，为第 3 大的值。
```

**示例 4：**

```
输入：matrix = [[5,2],[1,6]], k = 4
输出：0
解释：坐标 (1,1) 的值是 5 XOR 2 XOR 1 XOR 6 = 0 ，为第 4 大的值。
```



做了好久的异或的问题，现在每当看到异或问题，就自然而然的想到了 ```前缀和```，何谓```前缀和```?

前缀和是一个数组的某项下标之前(包括此项元素)的所有数组元素的和。

设b[]为前缀和数组，a[]为原数组，根据这句话可以得到前缀和的定义式和递推式：

|            | 定义式                                                       | 递推式                                                       |
| ---------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 一维前缀和 | ![b[i]=\sum_{j=0}^{i}a[j]](https://private.codecogs.com/gif.latex?b%5Bi%5D%3D%5Csum_%7Bj%3D0%7D%5E%7Bi%7Da%5Bj%5D) | ![b[i]=b[i-1]+a[i]](https://private.codecogs.com/gif.latex?b%5Bi%5D%3Db%5Bi-1%5D&plus;a%5Bi%5D) |
| 二维前缀和 | ![b[x][y]=\sum_{i=0}^{x}\sum_{j=0}^{y}a[i][j]](https://private.codecogs.com/gif.latex?b%5Bx%5D%5By%5D%3D%5Csum_%7Bi%3D0%7D%5E%7Bx%7D%5Csum_%7Bj%3D0%7D%5E%7By%7Da%5Bi%5D%5Bj%5D) | ![b[x][y]=b[x-1][y]+b[x][y-1]-b[x-1][y-1]+a[x][y]](https://private.codecogs.com/gif.latex?b%5Bx%5D%5By%5D%3Db%5Bx-1%5D%5By%5D&plus;b%5Bx%5D%5By-1%5D-b%5Bx-1%5D%5By-1%5D&plus;a%5Bx%5D%5By%5D) |

 

【一维前缀和】

根据上面的定义，我们可以很容易得到 sum[i] = sum[i-1] + a[i] 　这样就可以得到前i个数的和

根据上述表达式我们可以以O(1)求出区间[i,j]的区间和   ![sum[i,j]=b[j]-b[i-1]](https://private.codecogs.com/gif.latex?sum%5Bi%2Cj%5D%3Db%5Bj%5D-b%5Bi-1%5D)

![](https://private.codecogs.com/gif.latex?sum%5Bi%2Cj%5D%3Db%5Bj%5D-b%5Bi-1%5D)



根据以上讲解，我们知道本题就是一道前缀和的问题，所以我们套用二维前缀和的公式即可解决:

```java
public static int kthLargestValue(int[][] matrix, int k) {
    int len = matrix.length;
    int wid = matrix[0].length;
    // 遍历这个数组，保存到中间数组中
    int[][] pre = new int[len + 1][wid + 1];
    List<Integer> res = new ArrayList();
    for (int i = 1; i <= len; i++) {
        for (int j = 1; j <= wid; j++) {
            pre[i][j] = pre[i - 1][j] ^ pre[i][j - 1] ^ pre[i - 1][j - 1] ^ matrix[i - 1][j - 1];
            res.add(pre[i][j]);
        }
    }
    // 然后进行排序
    Collections.sort(res, new Comparator<Integer>() {
        @Override
        public int compare(Integer o1, Integer o2) {
            return o2 - o1;
        }
    });
    return res.get(k -1);
}
```

