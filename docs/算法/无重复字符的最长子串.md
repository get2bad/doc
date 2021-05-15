# [无重复字符的最长子串](https://leetcode-cn.com/problems/longest-substring-without-repeating-characters/)

给定一个字符串，请你找出其中不含有重复字符的 **最长子串** 的长度。

 

**示例 1:**

```
输入: s = "abcabcbb"
输出: 3 
解释: 因为无重复字符的最长子串是 "abc"，所以其长度为 3。
```

**示例 2:**

```
输入: s = "bbbbb"
输出: 1
解释: 因为无重复字符的最长子串是 "b"，所以其长度为 1。
```

**示例 3:**

```
输入: s = "pwwkew"
输出: 3
解释: 因为无重复字符的最长子串是 "wke"，所以其长度为 3。
     请注意，你的答案必须是 子串 的长度，"pwke" 是一个子序列，不是子串。
```

**示例 4:**

```
输入: s = ""
输出: 0
```



## 相同子串

看到这道题，第一想法就是使用快慢指针法，我们可以很灵活的解出本题。

最后发现，尴尬了！我求的是相同字符的最长长度。。。。。。

```java
public static int lengthOfLongestSubstring(String s) {
    if(s.trim().length() == 0) return 0;
    int res = 0,slow = 0;
    char pre = ' ';
    for (int fast = 1; fast < s.length(); fast++) {
        if(s.charAt(slow) == s.charAt(fast) && s.charAt(slow) != pre){
            // 如果相等
            if(isSubString(s,slow,fast)){
                res += fast - slow;
                slow = fast;
                pre = s.charAt(fast);
            }
        }
    }
    return res;
}
public static boolean isSubString(String s,int start,int end){
    int source = end;
    while(start < source){
        if(s.charAt(start++) != s.charAt(end ++)){
            return false;
        }
    }
    return true;
}
```

## 不同子串

转换思路，进行下一步求不同的尝试，发现我们可以使用HashMap数据结构来搞定

大体思路是：

使用快慢指针(滑动窗口)的思想来解决本题：

1. 声明一个慢指针slow 指向字符串的第一个元素，声明一个HashMap存储字符串的每个元素
2. 遍历这个字符串
   1. 如果HashMap中存在当前遍历的快指针元素，那么就要调整慢指针到重复元素的下一个索引的位置
   2. 添加当前遍历元素到HashMap中，如果之前存在，根据HashMap的特性，会自动覆盖之前的值
   3. 更新返回的max值为 从 (之前的 max值 和 （快指针 - 慢指针 + 1）) 中取最大值

下面说说一个比较特殊的例子的解题过程( pwwkew ) 主要解析for循环中的过程

1. source is empty max = 0 slow = 0 fast = 0  
2.  Source = {{'p',0}}  max = 1 slow = 0 fast = 0
3. source = {{'p',0},{'w',1}} max = 2 slow = 0 fast = 1
4. source = {{'p',0},{'w',2}} max = 2 slow = 2 fast = 2
5. Source = {{'p',0},{'w',2},{'k',3}} max = 2 slow = 2 fast = 3
6. Source = {{'p',0},{'w',2},{'k',3},{'e',4}} max = 3 slow = 2 fast = 4
7. source =  {{'p',0},{'w',5},{'k',3},{'e',4}} max = 3 slow = 3 fast = 5
8. 返回 3

```java
public static int lengthOfLongestSubstring1(String s) {
    if(s.length() == 0) return 0;
    Map<Character,Integer> source = new HashMap<>();
    int max = 0,slow = 0;
    for (int fast = 0; fast < s.length(); fast++) {
        if(source.containsKey(s.charAt(fast))){
            // 如果有，就调整慢指针到当前遍历元素的下一个指针位置
            slow = Math.max(slow,source.get(s.charAt(fast)) + 1);
        }
        // 将当前遍历的元素其添加进去
        source.put(s.charAt(fast),fast);
        max = Math.max(max,fast - slow + 1);
    }
    return max;
}
```



