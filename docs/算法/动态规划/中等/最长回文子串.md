# [最长回文子串](https://leetcode-cn.com/problems/longest-palindromic-substring/)

给你一个字符串 `s`，找到 `s` 中最长的回文子串。

**示例 1：**

```
输入：s = "babad"
输出："bab"
解释："aba" 同样是符合题意的答案。
```

**示例 2：**

```
输入：s = "cbbd"
输出："bb"
```

**示例 3：**

```
输入：s = "a"
输出："a"
```

**示例 4：**

```
输入：s = "ac"
输出："a"
```

**示例 5：**

```
输入：s = "abbacd"
输出："abba"
```

## 动态规划

> 中心扩散的方法，其实做了很多重复计算。动态规划就是为了减少重复计算的问题。动态规划听起来很高大上。其实说白了就是空间换时间，将计算结果暂存起来，避免重复计算。作用和工程中用 `redis` 做缓存有异曲同工之妙。
> 我们用一个 `boolean dp[l][r]` 表示字符串从 `i` 到 `j` 这段是否为回文。试想如果 `dp[l][r]=true`，我们要判断 `dp[l-1][r+1]` 是否为回文。只需要判断字符串在(`l-1`)和（`r+1`)两个位置是否为相同的字符，是不是减少了很多重复计算。
> 进入正题，<font color=red>动态规划关键是找到初始状态和状态转移方程。</font>
> <font color=red>初始状态，`l=r` 时，此时 `dp[l][r]=true`。
> 状态转移方程，`dp[l][r]=true` 并且(`l-1`)和（`r+1`)两个位置为相同的字符，此时 `dp[l-1][r+1]=true`。</font>

```java
public String longestPalindrome(String s) {
    if (s == null || s.length() < 2) {
        return s;
    }
    int strLen = s.length();
    int maxStart = 0;  //最长回文串的起点
    int maxEnd = 0;    //最长回文串的终点
    int maxLen = 1;  //最长回文串的长度

    boolean[][] dp = new boolean[strLen][strLen];

    for (int r = 1; r < strLen; r++) {
        for (int l = 0; l < r; l++) {
            if (s.charAt(l) == s.charAt(r) && (r - l <= 2 || dp[l + 1][r - 1])) {
                dp[l][r] = true;
                if (r - l + 1 > maxLen) {
                    maxLen = r - l + 1;
                    maxStart = l;
                    maxEnd = r;

                }
            }
        }
    }
    return s.substring(maxStart, maxEnd + 1);

}
```





## 小补充： 中心扩散法

> 中心扩散法怎么去找回文串？
> 从每一个位置出发，向两边扩散即可。遇到不是回文的时候结束。举个例子，str = acdbbdaa*s**t**r*=*a**c**d**b**b**d**a**a* 我们需要寻找从第一个 `b`（位置为 33）出发最长回文串为多少。怎么寻找？
> 首先往左寻找与当期位置相同的字符，直到遇到不相等为止。
> 然后往右寻找与当期位置相同的字符，直到遇到不相等为止。
>
> 最后左右双向扩散，直到左和右不相等。如下图所示：
>
> ![](https://pic.leetcode-cn.com/2f205fcd0493818129e8d3604b2d84d94678fda7708c0e9831f192e21abb1f34.png)

```java
public String longestPalindrome1(String s) {
    if (s == null || s.length() == 0) {
        return "";
    }
    int strLen = s.length();
    int left = 0;
    int right = 0;
    int len = 1;
    int maxStart = 0;
    int maxLen = 0;

    for (int i = 0; i < strLen; i++) {
        left = i - 1;
        right = i + 1;
        while (left >= 0 && s.charAt(left) == s.charAt(i)) {
            len++;
            left--;
        }
        while (right < strLen && s.charAt(right) == s.charAt(i)) {
            len++;
            right++;
        }
        while (left >= 0 && right < strLen && s.charAt(right) == s.charAt(left)) {
            len = len + 2;
            left--;
            right++;
        }
        if (len > maxLen) {
            maxLen = len;
            maxStart = left;
        }
        len = 1;
    }
    return s.substring(maxStart + 1, maxStart + maxLen + 1);

}
```



## 小补充： 暴力枚举

```java
public boolean isPalindromic(String s) {
	int len = s.length();
	for (int i = 0; i < len / 2; i++) {
		if (s.charAt(i) != s.charAt(len - i - 1)) {
			return false;
		}
	}
	return true;
}

// 暴力解法
public String longestPalindrome(String s) {
    String ans = "";
    int max = 0;
    int len = s.length();
    for (int i = 0; i < len; i++)
    	for (int j = i + 1; j <= len; j++) {
    	    String test = s.substring(i, j);
    	    if (isPalindromic(test) && test.length() > max) {
    	        ans = s.substring(i, j);
    	        max = Math.max(max, ans.length());
    	 		}
    	}
    return ans;
}
```

