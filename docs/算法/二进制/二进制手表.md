# [二进制手表](https://leetcode-cn.com/problems/binary-watch/)

二进制手表顶部有 4 个 LED 代表 **小时（0-11）**，底部的 6 个 LED 代表 **分钟（0-59）**。每个 LED 代表一个 0 或 1，最低位在右侧。

- 例如，下面的二进制手表读取 `"3:25"` 。

<img src="http://image.tinx.top/20210621184216.png" style="zoom: 50%;" />

*（图源：[WikiMedia - Binary clock samui moon.jpg](https://commons.m.wikimedia.org/wiki/File:Binary_clock_samui_moon.jpg) ，许可协议：[Attribution-ShareAlike 3.0 Unported (CC BY-SA 3.0)](https://creativecommons.org/licenses/by-sa/3.0/deed.en) ）*

给你一个整数 `turnedOn` ，表示当前亮着的 LED 的数量，返回二进制手表可以表示的所有可能时间。你可以 **按任意顺序** 返回答案。

小时不会以零开头：

- 例如，`"01:00"` 是无效的时间，正确的写法应该是 `"1:00"` 。

分钟必须由两位数组成，可能会以零开头：

- 例如，`"10:2"` 是无效的时间，正确的写法应该是 `"10:02"` 。

**示例 1：**

```
输入：turnedOn = 1
输出：["0:01","0:02","0:04","0:08","0:16","0:32","1:00","2:00","4:00","8:00"]
```

**示例 2：**

```
输入：turnedOn = 9
输出：[]
```



## 笨方法 (哈希表法)

思路：

说起思路！第一想到的就是 使用 Map<Integer,String> 来搞定，因为每一个时间都有代表亮起的灯的数量，然后我们只要对求和加起来的和turnedOn一致即可,但是本解决方案超级繁琐

```java
// 代码不做过多展示，脑壳痛，自行脑补
```



## 笨方法(暴力法)

思路：

此方法比较简单，直接两层for循环，第一层是小时，第二层是分钟，然后计算每一次的二进制位1的个数，两个加起来就是需要返回的结果

```java
public static List<String> readBinaryWatch(int turnedOn) {
    List<String> res = new ArrayList<>();
    if(turnedOn >= 8) return Arrays.asList();
    for (int i = 0; i < 12; i++) {
        for (int j = 0; j < 60; j++) {
            if(getOneCnt(i) + getOneCnt(j) == turnedOn){
                res.add(i + ":" + (j < 10? "0"+j : j));
            }
        }
    }
    return res;
}

public static int getOneCnt(Integer target){
    int res = 0;
    while (target != 0) {
        target = target & (target - 1);
        res++;
    }
    return res;
}
```





## 数学法(毫无规律可言，放弃！)

思路：

1 - 12点 亮起的灯数： 1，1，2，1，2，2，3，1，2，2，3，2

00分到59分亮起的灯数：0，1，1，2，1，2，2，3，1，2，2，3，2，3，2，3，1，2，2，3，2，3，3



## 二进制枚举

阅读的答案，答案的思路是：(是真的没有想到)

另一种枚举方法是枚举所有 2^10=1024 种灯的开闭组合，即用一个二进制数表示灯的开闭，其高 4位为小时，低 6 位为分钟。若小时和分钟的值均在合法范围内，且二进制中 1 的个数为 turnedOn，则将其加入到答案中。

```java
public static List<String> readBinaryWatch(int turnedOn) {
    List<String> ans = new ArrayList<String>();
    for (int i = 0; i < 1024; ++i) {
        int h = i >> 6, m = i & 63; // 用位运算取出高 4 位和低 6 位
        if (h < 12 && m < 60 && Integer.bitCount(i) == turnedOn) {
            ans.add(h + ":" + (m < 10 ? "0" : "") + m);
        }
    }
    return ans;
}
```

