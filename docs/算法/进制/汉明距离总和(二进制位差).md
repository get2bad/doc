# [汉明距离总和](https://leetcode-cn.com/problems/total-hamming-distance/)

两个整数的 [汉明距离](https://baike.baidu.com/item/汉明距离/475174?fr=aladdin) 指的是这两个数字的二进制数对应位不同的数量。

计算一个数组中，任意两个数之间汉明距离的总和。

**示例:**

```
输入: 4, 14, 2

输出: 6

解释: 在二进制表示中，4表示为0100，14表示为1110，2表示为0010。（这样表示是为了体现后四位之间关系）
所以答案为：
HammingDistance(4, 14) + HammingDistance(4, 2) + HammingDistance(14, 2) = 2 + 2 + 2 = 6.
```



其实解决本题很简单，就是看每一位的和下一个的二进制位的差和，刚接触到本题很容易就想到了使用暴力法，即双层for循环的方式：

## 暴力法

```java
public static int totalHammingDistance(int[] nums) {
    int res = 0;
    for(int i = 0; i < nums.length - 1; i++){
        for(int j = i+1;j < nums.length; j++){
            res += getHamm(nums[i],nums[j]);
        }
    }
    return res;
}
private static int getHamm(int x,int y){
    return Integer.bitCount(x^y);
}
```



但是答案列举了以下的方式，很巧妙：

## 逐位计算法

本题的大体思路就是，找到32位每一位与1进行与运算：

+ 如果当前位是1，和1进行与运算，那么结果就是1，将s1进行加1操作
+ 如果当前位是0，和1进行与运算，那么记过就是0，将s0进行加1操作

**要求得「第 x位所有不同数」的对数的个数，只需要应用乘法原理，将两者元素个数相乘即可**

下面，我们参考[宫水三叶](https://leetcode-cn.com/u/ac_oier/)大姐头提供的图：

![](http://image.tinx.top/20210528114032.png)

```java
public int totalHammingDistance1(int[] nums) {
    int ans = 0;
    for (int x = 31; x >= 0; x--) {
        int s0 = 0, s1 = 0;
        for (int u : nums) {
            if (((u >> x) & 1) == 1) {
                s1++;
            } else {
                s0++;
            }  
        }
        ans += s0 * s1;
    }
    return ans;
}
```

