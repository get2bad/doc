# [面试题 17.21. 直方图的水量](https://leetcode-cn.com/problems/volume-of-histogram-lcci/)

给定一个直方图(也称柱状图)，假设有人从上面源源不断地倒水，最后直方图能存多少水量?直方图的宽度为 1。

![img](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2018/10/22/rainwatertrap.png)

上面是由数组 [0,1,0,2,1,0,1,3,2,1,2,1] 表示的直方图，在这种情况下，可以接 6 个单位的水（蓝色部分表示水）。

**示例:**

```
输入: [0,1,0,2,1,0,1,3,2,1,2,1]
输出: 6
```



思考：

本题的大体意思就是：

蓝色表示水，黑色表示容器，输入的数组就是容器的量，计算能装多少水的的关键就是遍历这个数组

如果当前元素的高度大于后面的遍历元素，那么就可以进行装水，一旦超过当前元素的高度说明装水停止，后面继续，其实我们可以进行```双重遍历```的方法来搞定本题



## 第一版答案(错误❌版本)

以下是我的双重遍历 版本，但是遇到一个小问题，就是终止条件那边有点小困难，并且如果当前遍历的元素一直没有与他相匹配的高度和大于他的高度，那么就一直加下去嘛？这肯定结果是不对的，感觉我的 ```i = j``` 这种 其实可以演变为 ```双指针```法

```java
// 0 2 0错误
public static int trap(int[] height) {
    if (height == null || height.length < 2) return 0;
    int res = 0;
    for (int i = 0; i < height.length; i++) {
        for (int j = i + 1; j < height.length; j++) {
            if (height[i] > height[j]) {
                res += height[i] - height[j];
                i = j;
            } else {
                break;
            }
        }
    }

    return res;
}
```



## 第二版本答案(正确✔️版本)

<font color=red>建议先看动态规划算法，本版本是动态规划法的优化版，节省了空间复杂度 O(n) -> O(1)</font>

大体的思路就是:

先记住之前元素的最大值，然后依次减去之后比它小的数，得到了可以接的雨水书数

```java
public static int trap1(int[] height) {
    int res = 0;
    int left = 0, right = height.length - 1;
    int leftMax = 0, rightMax = 0;
    while (left < right) {
        leftMax = Math.max(leftMax, height[left]);
        rightMax = Math.max(rightMax, height[right]);
        if (height[left] < height[right]) {
            res += leftMax - height[left];
            left++;
        } else {
            res += rightMax - height[right];
            right--;
        }
    }
    return res;
}
```



## 动态规划算法(摘抄自答案)

一张图解决你所有的疑惑

![](https://assets.leetcode-cn.com/solution-static/jindian_17.21/1.png)



上图的意思就是分别正向 / 逆向取得最高高度的染色，然后将 正向 / 逆向 重叠，重合的部分就是接雨量

```java
class Solution {
    public int trap(int[] height) {
        int n = height.length;
        if (n == 0) {
            return 0;
        }

        int[] leftMax = new int[n];
        leftMax[0] = height[0];
        for (int i = 1; i < n; ++i) {
            leftMax[i] = Math.max(leftMax[i - 1], height[i]);
        }

        int[] rightMax = new int[n];
        rightMax[n - 1] = height[n - 1];
        for (int i = n - 2; i >= 0; --i) {
            rightMax[i] = Math.max(rightMax[i + 1], height[i]);
        }

        int ans = 0;
        for (int i = 0; i < n; ++i) {
            ans += Math.min(leftMax[i], rightMax[i]) - height[i];
        }
        return ans;
    }
}
```

