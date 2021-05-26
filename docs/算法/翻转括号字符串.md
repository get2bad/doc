# [反转每对括号间的子串](https://leetcode-cn.com/problems/reverse-substrings-between-each-pair-of-parentheses/)

给出一个字符串 `s`（仅含有小写英文字母和括号）。

请你按照从括号内到外的顺序，逐层反转每对匹配括号中的字符串，并返回最终的结果。

注意，您的结果中 **不应** 包含任何括号。

**示例 1：**

```
输入：s = "(abcd)"
输出："dcba"
```

**示例 2：**

```
输入：s = "(u(love)i)"
输出："iloveu"
```

**示例 3：**

```
输入：s = "(ed(et(oc))el)"
输出："leetcode"
```

**示例 4：**

```
输入：s = "a(bcdefghijkl(mno)p)q"
输出："apmnolkjihgfedcbq"
```



刚看到这道题的时候，发现应该是字符串倒装，去掉括号()，于是就使用 

```string.replace("(","").replace(")","").reverse()```

发现并没有那么简单！

通读示例后，发现大致原理就是：

遇到( 左括号后，就会去找)右括号，然后从那边开始倒装，一次类推，根据这个特性，我们直接使用栈来解决本题：

```java
public static String reverseParentheses(String s) {
    StringBuilder sb = new StringBuilder();
    char[] arr = s.toCharArray();
    Stack<Integer> stack = new Stack<>();

    for (int i = 0; i < arr.length; i++) {
        if (arr[i] == '(')
            stack.push(i);
        if (arr[i] == ')')
            reverse(arr, stack.pop() + 1, i - 1);
    }

    for (int i = 0; i < arr.length; i++)
        if (arr[i] != ')' && arr[i] != '(')
            sb.append(arr[i]);

    return sb.toString();
}

public static void reverse(char[] arr, int left, int right) {
    while (right > left) {
        char tmp = arr[left];
        arr[left] = arr[right];
        arr[right] = tmp;
        right--;
        left++;
    }
}
```

看答案后，发现还有一种更简单的方法：

大体步骤讲解：

如果遍历遇到了左括号(，那么就将叠加的字符串放进栈中(栈的意义就是来回reverse);

如果遍历遇到了右括号)，那么就将临时字符串翻转，然后将栈中的元素出栈，加入到临时字符串头的位置

> 碰到了右括号，说明翻转开始的标志，所以直接将之前的东西reverse，然后将栈中暂存的东西添加到临时字符串头部

如果没有碰到括号，那么就向临时字符串尾部添加现在的遍历元素

最终返回临时字符串

```java
public static String reverseParentheses1(String s) {
    Deque<String> stack = new LinkedList<String>();
    StringBuffer sb = new StringBuffer();
    for (int i = 0; i < s.length(); i++) {
        char ch = s.charAt(i);
        if (ch == '(') {
            stack.push(sb.toString());
            sb.setLength(0);
        } else if (ch == ')') {
            sb.reverse();
            sb.insert(0, stack.pop());
        } else {
            sb.append(ch);
        }
    }
    return sb.toString();
}
```

