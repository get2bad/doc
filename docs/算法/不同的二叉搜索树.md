# [96. 不同的二叉搜索树](https://leetcode-cn.com/problems/unique-binary-search-trees/)

给定一个整数 *n*，求以 1 ... *n* 为节点组成的二叉搜索树有多少种？

**示例:**

```
输入: 3
输出: 5
解释:
给定 n = 3, 一共有 5 种不同结构的二叉搜索树:

   1         3     3      2      1
    \       /     /      / \      \
     3     2     1      1   3      2
    /     /       \                 \
   2     1         2                 3
```



动态规划专题 - 不同的二叉搜索树

+ 初始状态

  ```java
  int[] dp = new int[n+1];
  dp[0] = 1;
  dp[1] = 1;
  ```

+ 状态转移公式([卡特兰数](https://baike.baidu.com/item/%E5%8D%A1%E7%89%B9%E5%85%B0%E6%95%B0))

  ```java
  // 卡特兰数
  dp[i] += dp[j-1] * dp[i-j];
  ```

  >  卡特兰数Cn满足以下递推关系 [2] ：
  >
  > ![img](https://bkimg.cdn.bcebos.com/formula/db8efb06fafd1311237a2e947de0ce8f.svg)
  >
  > ![img](https://bkimg.cdn.bcebos.com/formula/4aae3d540ed3bfee135b7c3a70be66b8.svg)

  输入2的图解：

![](https://pic.leetcode-cn.com/96bb714d079c7aef72465216b1f205cbf78865f6bcc3cf69691f7d46096196e0-frame_00010.png)

输入3的图解：

![](https://pic.leetcode-cn.com/69509ce7a83c6c57bc6fb611db21ef1a83c552cc4237e75fb338162c8e128c12-frame_00016.png)

```java
public static int numTrees(int n) {
    int[] res = new int[n + 1];
    // 如果是输入 0 或者 1 的话 他们只有一种方法
    res[0] = 1;
    res[1] = 1;

    for (int i = 2; i <= n; ++i) {
        for (int j = 1; j <= i; ++j) {
            res[i] += res[j - 1] * res[i - j];
        }
    }
    return res[n];
}
```











