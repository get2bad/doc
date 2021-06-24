# [搜索二维矩阵](https://leetcode-cn.com/problems/search-a-2d-matrix/)

编写一个高效的算法来判断 `m x n` 矩阵中，是否存在一个目标值。该矩阵具有如下特性：

- 每行中的整数从左到右按升序排列。
- 每行的第一个整数大于前一行的最后一个整数。

 

**示例 1：**

![img](https://assets.leetcode.com/uploads/2020/10/05/mat.jpg)

```
输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 3
输出：true
```

**示例 2：**

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2020/11/25/mat2.jpg)

```
输入：matrix = [[1,3,5,7],[10,11,16,20],[23,30,34,60]], target = 13
输出：false
```



看上面的内容，这不就是一个高级版的数值查找算法嘛



我感觉可以使用 变种版的 **折半查询**

大体思路是这样：

1. 首先遍历这个二维数组，先遍历行，然后遍历列
2. 在遍历列的时候 如果当前 行首 小于 目标值， 行末 大于目标值，就开始进行查找
3. 然后进行 **折半查询** 即可 完成本题！

```java
public static boolean searchMatrix(int[][] matrix, int target) {
    if(matrix == null) return false;
    int length = matrix.length;
    int width = matrix[0].length;
    for (int i = 0; i < length; i++) {
        for (int j = 0; j < width; j++) {
            if(matrix[i][0] <= target && matrix[i][width - 1] >= target){
                // 进行二分查找
                return search(matrix[i],target);
            }
        }
    }
    return false;
}
// 二分查找
public static boolean search(int[] source, int target){
    int start = 0, end = source.length - 1;
    int middle = 0;
    while(end >= start){
        middle = (end + start) / 2;
        if(source[middle] > target){
            end = middle - 1;
        } else if(source[middle] < target){
            start = middle + 1;
        } else {
            return true;
        }
    }
    return false;
}
```



对于这种简单题，我直接重拳出击，哎，无敌的寂寞~