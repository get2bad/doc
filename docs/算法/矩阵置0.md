# [矩阵置零](https://leetcode-cn.com/problems/set-matrix-zeroes/)

给定一个 `*m* x *n*` 的矩阵，如果一个元素为 **0** ，则将其所在行和列的所有元素都设为 **0** 。请使用 **[原地](http://baike.baidu.com/item/原地算法)** 算法**。

 

**示例 1：**

![img](https://assets.leetcode.com/uploads/2020/08/17/mat1.jpg)

```
输入：matrix = [[1,1,1],[1,0,1],[1,1,1]]
输出：[[1,0,1],[0,0,0],[1,0,1]]
```

**示例 2：**

![img](https://assets.leetcode.com/uploads/2020/08/17/mat2.jpg)

```
输入：matrix = [[0,1,2,0],[3,4,5,2],[1,3,1,5]]
输出：[[0,0,0,0],[0,4,5,0],[0,3,1,0]]
```



本题的难点就是在于，遇到0的时候，之前的元素如何处理，因为进阶条件没有限定死 时间复杂度，只是限定了空间复杂度，要小于 O(m +n)，关于之前的元素置 0 的问题，我们可以用 

## **标记数组法** 

但是空间复杂度为 O(m+n)，比较巧妙的保存了当前置位0的元素，进行二次遍历，然后置 0 

```java
 /**
 * 空间复杂度 O(m + n)的解法
 * 因为要将行和列周围的数字都置位0
 * 所以：
 * 假设 source[1][1]的元素为0
 * 那么置位0的元素应该是
 *          source[1][x]
 *          source[x][1]
 *  那么我们可以用标记数组法，进行两次for循环
 *  时间复杂度应该为O(mn)
 */
public static void setZeroes(int[][] matrix) {
    int length = matrix.length;
    int width = matrix[0].length;
    // 表示行
    boolean[] lon = new boolean[length];
    // 表示列
    boolean[] wid = new boolean[width];
    // 找到元素为0 的位置
    for (int i = 0; i < length; i++) {
        for (int j = 0; j < width; j++) {
            if(matrix[i][j] == 0){
                lon[i] = wid[j] = true;
            }
        }
    }
    // 遍历这个标记数组，然后遇到0 那么我们就将其置位0
    for (int i = 0; i < length; i++) {
        for (int j = 0; j < width; j++) {
            if( lon[i] || wid[j] ){
                matrix[i][j] = 0;
            }
        }
    }
}
```



## 使用一个标记变量法

仅仅使用一个标记变量,大概的思路是这样的：

1. 先遍历一下这个目标数组的行，

2. 如果行的第一个元素为0 就将 flag 置为 true

3. 然后继续遍历数组的列，如果当前遍历的元素martix`[i][j]` = 0 那么就将其同行 / 同列元素都变为0

   > Martix = [1,2,3],				martix = [1,0,3],
   >
   > ​				[4,0,5],		=>					[0,0,5],
   >
   > ​				[6,7,8]								[6,7,8]

4. <font color=red>上述步骤相当于把之前的元素全部置为 0</font> ，接下来就是后面的元素了，所以第二次遍历，我们使用<font color=red>```行的倒序遍历```</font>

   > 因为行的前序遍历的话，之前置为0 的元素会**污染**后面的元素，这样数据就不正确了
   >
   > 比如下面这个矩阵
   >
   > ```scss
   > [0,1,2,0],
   > [3,4,5,2],
   > [1,3,1,5]
   > ```
   >
   > 按照算法三, 我们之所以要从后面开始处理, 是因为如果我们先处理了第一行那么矩阵变成了
   >
   > ```scss
   >  [0,0,0,0],
   >  [3,4,5,2],
   >  [1,3,1,5]
   > 第一行的第二列和第三列的0元素影响了下面的 4, 5 和3, 1
   > 但是其实这两个0不是本来存在的, 而是处理之后出现的
   > 所以我们从下面处理, 就不会被这两个0影响了.
   > ```

5. 如果当前的遍历元素 行 / 列的第一个元素为0 ，那么我们就将当前遍历的元素置为 0 

   > Martix = [1,0,3],				martix = [1,0,3],
   >
   > ​				[0,0,5],		=>					[0,0,0],
   >
   > ​				[6,7,8]								[6,0,8]

6. 如果我们之前将flag 置位 true 的是因为行首元素为 0 ，所以我们之前的操作时，每一次遍历就要将行首置为0 要不之前的没有讲行首置为0 的操作，会导致结果错误

```java
public static void setZeroes1(int[][] matrix) {
    int length = matrix.length;
    int width = matrix[0].length;
    boolean flag = false;
    for (int i = 0; i < length; i++) {
        if (matrix[i][0] == 0) flag = true;
        for (int j = 1; j < width; j++) {
            if (matrix[i][j] == 0) {
                matrix[i][0] = matrix[0][j] = 0;
            }
        }
    }

    for (int i = length - 1; i >= 0; i--) {
        for (int j = 1; j < width; j++) {
            if (matrix[i][0] == 0 || matrix[0][j] == 0) {
                matrix[i][j] = 0;
            }
        }
        if (flag) matrix[i][0] = 0;
    }
}
```

