# [ 同构字符串](https://leetcode-cn.com/problems/isomorphic-strings/)

给定两个字符串 ***s*** 和 **t**，判断它们是否是同构的。

如果 ***s*** 中的字符可以按某种映射关系替换得到 **t** ，那么这两个字符串是同构的。

每个出现的字符都应当映射到另一个字符，同时不改变字符的顺序。不同字符不能映射到同一个字符上，相同字符只能映射到同一个字符上，字符可以映射到自己本身。

 

**示例 1:**

```
输入：s = "egg", t = "add"
输出：true
```

**示例 2：**

```
输入：s = "foo", t = "bar"
输出：false
```

**示例 3：**

```
输入：s = "paper", t = "title"
输出：true
```



起初做本题的时候，看到题目要求，还以为是ASCII码运算，然后看每一位是否按照某种规律做了映射，就放到结果中运行，发现答案不正确

```java
public boolean isIsomorphic(String s, String t) {
    if(s.length() != t.length()) return false;
    Integer tmp = Integer.MAX_VALUE;
    for (int i = 0; i < s.length(); i++) {
        char[] pre = s.toCharArray();
        char[] after = t.toCharArray();
        int source = pre[i] - after[i];
        if(tmp == Integer.MAX_VALUE){
            tmp = source;
        }
        if(source != tmp){
            return false;
        }
    }
    return true;
}
```

然后感觉不可能啊，就看了看答案，顿时无语，他指的映射就是如果之前存在的，那么映射的不对才会输出false....

既然这样，那就更加简单了，我们可以使用 哈希表来解出本题：

```java
/**
 * 哈希表 存储映射关系
 */
public static boolean isIsomorphic1(String s, String t) {
    if(s.length() != t.length()) return false;
    char[] pre = s.toCharArray();
    char[] after = t.toCharArray();
    Map<Character,Character> s1 = new HashMap<>();
    Map<Character,Character> t1 = new HashMap<>();
    for (int i = 0; i < s.length(); i++) {
        if ((s1.containsKey(pre[i]) && s1.get(pre[i]) != after[i])
                || (t1.containsKey(after[i]) && t1.get(after[i]) != pre[i])){
            return false;
        }
        s1.put(pre[i],after[i]);
        t1.put(after[i],pre[i]);
    }
    return true;
}
```

