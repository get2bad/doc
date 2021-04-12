# [判断子序列](https://leetcode-cn.com/problems/is-subsequence/)

给定字符串 s 和 t ，判断 s 是否为 t 的子序列。

字符串的一个子序列是原始字符串删除一些（也可以不删除）字符而不改变剩余字符相对位置形成的新字符串。（例如，"ace"是"abcde"的一个子序列，而"aec"不是）。

进阶：

如果有大量输入的 S，称作 S1, S2, ... , Sk 其中 k >= 10亿，你需要依次检查它们是否为 T 的子序列。在这种情况下，你会怎样改变代码？



示例 1：

输入：s = "abc", t = "ahbgdc"
输出：true
示例 2：

输入：s = "axc", t = "ahbgdc"
输出：false



本题有两种解题办法：

## 双指针法

```
/**
 * 双指针法
 * 核心思想就是，使用while遍历，遍历这两个字符串，如果遇到相同的情况，就将i++，每次遍历都叠加j
 *
 * 最终输出 两者是否相同即可
 */
public static boolean isSubsequence(String s, String t) {
    int n = s.length(), m = t.length();
    int i = 0, j = 0;
    while (i < n && j < m) {
        if (s.charAt(i) == t.charAt(j)) {
            i++;
        }
        j++;
    }
    return i == n;
}
```



## 动态规划法

动态规划的核心就是找到转移状态方程，这里的方程就是：

+ 如果相同

  ```res[i][j] = res[i - 1][j - 1] 和左上角的相同，因为本题的目标就是要求顺序不能乱，要从a开始 z结束，如果相同，那么左上角一定是 true```

+ 如果不同

  ```res[i][j] = res[i][j - 1]```

```
/**
     * 核心思想就是，如果当前遍历的字符等于需要查找的字符串，那么就将他左上角的值给现在的值(为了保证有序性)，如果不等于，就将他之前的值赋予
     * 例如：
     *      a   s   d   f   g   结果
     * t    t   t   t   t   t   t
     * a    t   t   t   t   t   t
     * d    f   f   t(左上角)  t   t   t
     * g    f   f   f   f   t(左上角)  t
     *
     * 最终输出结果即可
     */
    for (int i = 1; i < res.length; i++) {
        char ch1 = s.charAt(i - 1);
        for (int j = 1; j < res[0].length; j++) {
            char ch2 = t.charAt(j - 1);
            if(ch1 == ch2){
                res[i][j] = res[i - 1][j - 1];
            } else{
                res[i][j] = res[i][j - 1];
            }
        }
    }
    // 输出最后一行的最后一个结果
    return res[res.length - 1][res[0].length - 1];
}
```