# [直线上最多的点数](https://leetcode-cn.com/problems/max-points-on-a-line/)

给你一个数组 `points` ，其中 `points[i] = [xi, yi]` 表示 **X-Y** 平面上的一个点。求最多有多少个点在同一条直线上。

 

**示例 1：**

<img src="http://image.tinx.top/20210624092020.png" style="zoom:50%;" />

```
输入：points = [[1,1],[2,2],[3,3]]
输出：3
```

**示例 2：**

<img src="http://image.tinx.top/20210624092048.png" style="zoom:50%;" />

```
输入：points = [[1,1],[3,2],[5,3],[4,1],[2,3],[1,4]]
输出：4
```





## 哈希表法

其实这种问题的解决思路很简单，就是一个求起始点到结束点连成的直线的斜率的问题

>  固定一点, 找其他点和这个点组成直线, 统计他们的斜率!

那么难点就是找出一条条的点连成的直线,

起初，我想直接找出一整条的直线，使用的 ```快慢指针```、```分区法```都无果，但是仔细一想，不对啊，一整条的直线相同，那么一条直线是由不同的两个点组成的，他们的斜率都一样，那样的话，哈希表不就搞定了，所以的出来了思路：

1. 使用嵌套for循环遍历传入的这个二维数组

2. 找出遍历的两个点的斜率 k(这里使用 gcd算法来求出斜率)

   > 什么是 gcd算法？ gcd算法就是最大约数方法，这样求出来的就是斜率，比起来之前的更加精确，但是缺点就是比较两点差值相除的速度慢，用到了递归算法

3. 然后向哈希表中存入这个斜率的键，如果存在，就在其基础上 + 1
4. 最后与上一次的斜率max值进行比较，如果比他大，当前斜率的个数作为最大max值
5. 最后 比较 上一次大的嵌套for循环的结果进行比较，取最大值
6. 结果新鲜出炉！

代码：

```java
// 哈希表解决方案
public static int maxPoints(int[][] ps) {
    int len = ps.length;
    int res = 1;
    for (int i = 0; i < len; i++) {
        Map<String, Integer> map = new HashMap<>();
        // 由当前点 i 发出的直线所经过的最多点数量
        int max = 0;
        int x1 = ps[i][0], y1 = ps[i][1];
        for (int j = i + 1; j < len; j++) {
            int x2 = ps[j][0], y2 = ps[j][1];
            // 找出两个点的 差值
            int xDiff = x1 - x2, yDiff = y1 - y2;
            // k 为两个点组成直线的斜率
            int k = gcd(xDiff, yDiff);
            // 一个临时值的斜率 key
            String key = (xDiff / k) + "_" + (yDiff / k);
            // 将求出的两个点的斜率存入哈希表，如果没有就是从1开始，如果有就是之前的值 + 1
            map.put(key, map.getOrDefault(key, 0) + 1);
            // 和上一次斜率进行比较，如果比上一次斜率大，那么最大斜率就是当前斜率
            max = Math.max(max, map.get(key));
        }
        // 取当前哈希表存储的斜率和 res的最大值 作为最新的最大值
        res = Math.max(res, max + 1);
    }
    return res;
}

// 求出斜率
private static int gcd(int a, int b) {
    return b == 0 ? a : gcd(b, a % b);
}
```

