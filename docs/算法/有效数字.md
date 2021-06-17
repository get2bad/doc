#### [有效数字](https://leetcode-cn.com/problems/valid-number/)

**有效数字**（按顺序）可以分成以下几个部分：

1. 一个 **小数** 或者 **整数**
2. （可选）一个 `'e'` 或 `'E'` ，后面跟着一个 **整数**

**小数**（按顺序）可以分成以下几个部分：

1. （可选）一个符号字符（`'+'` 或 `'-'`）
2. 下述格式之一：
   1. 至少一位数字，后面跟着一个点 `'.'`
   2. 至少一位数字，后面跟着一个点 `'.'` ，后面再跟着至少一位数字
   3. 一个点 `'.'` ，后面跟着至少一位数字

**整数**（按顺序）可以分成以下几个部分：

1. （可选）一个符号字符（`'+'` 或 `'-'`）
2. 至少一位数字

部分有效数字列举如下：

- `["2", "0089", "-0.1", "+3.14", "4.", "-.9", "2e10", "-90E3", "3e+7", "+6e-1", "53.5e93", "-123.456e789"]`

部分无效数字列举如下：

- `["abc", "1a", "1e", "e3", "99e2.5", "--6", "-+3", "95a54e53"]`

给你一个字符串 `s` ，如果 `s` 是一个 **有效数字** ，请返回 `true` 。

**示例 1：**

```
输入：s = "0"
输出：true
```

**示例 2：**

```
输入：s = "e"
输出：false
```

**示例 3：**

```
输入：s = "."
输出：false
```

**示例 4：**

```
输入：s = ".1"
输出：true
```



通过审题以及相关的数学知识积累，我们可以知道分不同的情况：

若是以'e' / 'E'为分割的话：

+ 左边可以为整数或者浮点数
+ 右边必须为整数

其他情况：

+ 符号位只能出现在第一位，且符号位只允许出现一次
+ 最少最少要有一个数字
+ 小数点最多出现一次



看出这些情况来，那么我们其实有以下几种实现方法：

1. 正则表达式(最简单的解决办法，但是表达式不是很好写)

```java
public static boolean isNumber1(String s){
    String regex = "[+-]?(\\d+)?\\.?(?(1)\\d*|\\d+)([eE][+-]?\\d+)?";
    return Pattern.matches(regex,s);
}
```

2. 模拟法

这种解决方案比较繁琐，就是需要挨位去判断是否是正确合理的数字字符串,直接写到头大，费脑子...

```java
public static boolean isNumber(String s) {
    for (int i = 0; i < s.length(); i++) {
        char c = s.charAt(i);
        if (c == 'E' || c == 'e') {
            return (isDeci(0, i, s) || isInt(0, i, s)) && isInt(i + 1, s.length(), s);
        }
    }
    return isDeci(0, s.length(), s) || isInt(0, s.length(), s);
}

public static boolean isDeci(int l, int r, String s) {//判断是否为小数
    int flag = 0;
    boolean res = false;
    for (int i = l; i < r; i++) {
        char c = s.charAt(i);
        if (c == '+' || c == '-') {
            if (i != l) {
                return false;
            }
        } else if (c == '.') {
            if (flag == 0) {
                flag++;
            } else {
                return false;
            }
        } else if (c >= '0' && c <= '9') {
            res = true;
        } else {
            return false;
        }
    }
    return res;
}

public static boolean isInt(int l, int r, String s) {//判断是否为整数
    boolean res = false;
    for (int i = l; i < r; i++) {
        char c = s.charAt(i);
        if (c == '+' || c == '-') {
            if (i != l) {
                return false;
            }
        } else if (c >= '0' && c <= '9') {
            res = true;
        } else {
            return false;
        }
    }
    return res;
}
```

