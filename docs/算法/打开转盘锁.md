# [打开转盘锁](https://leetcode-cn.com/problems/open-the-lock/)

你有一个带有四个圆形拨轮的转盘锁。

每个拨轮都有10个数字： `'0', '1', '2', '3', '4', '5', '6', '7', '8', '9'` 。

每个拨轮可以自由旋转：例如把 `'9'` 变为 `'0'`，`'0'` 变为 `'9'` 。每次旋转都只能旋转一个拨轮的一位数字。

锁的初始数字为 `'0000'` ，一个代表四个拨轮的数字的字符串。

列表 `deadends` 包含了一组死亡数字，一旦拨轮的数字和列表里的任何一个元素相同，这个锁将会被永久锁定，无法再被旋转。

字符串 `target` 代表可以解锁的数字，你需要给出解锁需要的最小旋转次数，如果无论如何不能解锁，返回 `-1` 。

**示例 1:**

```
输入：deadends = ["0201","0101","0102","1212","2002"], target = "0202"
输出：6
解释：
可能的移动序列为 "0000" -> "1000" -> "1100" -> "1200" -> "1201" -> "1202" -> "0202"。
注意 "0000" -> "0001" -> "0002" -> "0102" -> "0202" 这样的序列是不能解锁的，
因为当拨动到 "0102" 时这个锁就会被锁定。
```

**示例 2:**

```
输入: deadends = ["8888"], target = "0009"
输出：1
解释：
把最后一位反向旋转一次即可 "0000" -> "0009"。
```

**示例 3:**

```
输入: deadends = ["8887","8889","8878","8898","8788","8988","7888","9888"], target = "8888"
输出：-1
解释：
无法旋转到目标数字且不被锁定。
```

**示例 4:**

```
输入: deadends = ["0000"], target = "8888"
输出：-1
```





思考：

刚阅读本题时，看到示例1的时候，心想直接 拨动 第二、四位的密码锁到2不就ok了么？仔细审题后，发现，只能从第一到第四依次拨动密码锁，以此类推，所以0202的拨动顺序为:

1. 因为deadends中含有0201，所以第一位不能不拨动，所以 要 1000
2. 1100
3. 1200
4. 1201
5. 1202
6. 0202



阅读清楚了题目，所以我们就可以编写以下代码来搞定问题：

好吧，我承认我是看答案的，菜鸡就是我，原本想的是 将target 和 当前转动的变成二进制，然后使用位运算来搞定，循环终止条件是  当前转动的大于等于target，不过又涉及到 deadends 问题，这种题目变得灵活多样，我的这种想法就变得有一些 幼稚了，之前做的题，实在想不出要如何解决了，故看了答案 ╮(╯▽╰)╭

本菜鸡的代码：

```java
public static int openLock(String[] deadends, String target) {
    int res = -1;
    String tar = Integer.toBinaryString(Integer.parseInt(target));
    String tmp = Integer.toBinaryString(0000);
    // 如果包含 目标密码 或者 0000 说明这个无论如何也无法解锁
    if(Arrays.asList(deadends).contains(target) || 
            Arrays.asList(deadends).contains("0000")){
        return -1;
    }
    // 开始转动密码锁
    while(Integer.parseInt(tmp) >= Integer.parseInt(tar)){
			// 逻辑处理，由于会碰到deadends导致循环体的代码异常难写，故放弃
    }

    return res;
}
```



## 双向BFS

他喵的，读个答案，都这么吃力，佛了佛了.....

大概的思路就是：

1. 声明正逆两个队列，正逆两个Map，队列用于存储上一次验证结束的字符串，Map用于存储当前验证字符串反转了几次
2. 开始循环，循环条件是如果两个队列都不为空的情况，方法体是哪个队列的容量小就遍历哪个队列，使之进行平均，如果有一个队列为空，说明从某个方向搜到底都搜不到该方向的目标节点
3. 遍历的规则就是 从第1位到第4位进行依次 + 1，遇到9就会逆转为 1，然后判断是否和 target相等，如果相等就直接返回 正向拨动次数 + 逆向拨动次数 + 1就是目标值

这就是双向(正逆)BFS

```java
public static int openLock(String[] deadends, String target) {
    s = "0000";
    t = target;
    if (s.equals(t)) {
        return 0;
    }

    for (String d : deadends) {
        set.add(d);
    }
    // 如果集合中存在，说明目标是锁死的，没法转动至目标的，所以返回-1
    if (set.contains(s)) {
        return -1;
    }
    // 拿到结果
    int ans = bfs();
    return ans;
}

private static int bfs() {
    // d1 代表从起点 s 开始搜索（正向）
    // d2 代表从结尾 t 开始搜索（反向）
    Deque<String> d1 = new ArrayDeque<>(), d2 = new ArrayDeque<>();
    /*
     * m1 和 m2 分别记录两个方向出现的状态是经过多少次转换而来
     * e.g.
     * m1 = {"1000":1} 代表 "1000" 由 s="0000" 替换 1 次字符而来
     * m2 = {"9999":3} 代表 "9999" 由 t="9996" 替换 3 次字符而来
     */
    Map<String, Integer> m1 = new HashMap<>(), m2 = new HashMap<>();
    // 正向入队
    d1.addLast(s);
    m1.put(s, 0);
    // 逆向入队
    d2.addLast(t);
    m2.put(t, 0);

    /*
     * 只有两个队列都不空，才有必要继续往下搜索
     * 如果其中一个队列空了，说明从某个方向搜到底都搜不到该方向的目标节点
     * e.g.
     * 例如，如果 d1 为空了，说明从 s 搜索到底都搜索不到 t，反向搜索也没必要进行了
     */
    while (!d1.isEmpty() && !d2.isEmpty()) {
        int t = -1;
        if (d1.size() <= d2.size()) {
            t = update(d1, m1, m2);
        } else {
            t = update(d2, m2, m1);
        }
        if (t != -1) return t;
    }
    return -1;
}

private static int update(Deque<String> deque, Map<String, Integer> cur, Map<String, Integer> other) {
    // 将队首出队，进行相关的处理
    String poll = deque.pollFirst();
    // 转换数组
    char[] pcs = poll.toCharArray();
    // 拿到这个 目标值转了密码盘几次
    int step = cur.get(poll);
    // 从 1 - 4 依次转数字
    for (int i = 0; i < 4; i++) {
        // 能「正向转」也能「反向转」，这里直接枚举偏移量 [-1,1] 然后跳过 0
        for (int j = -1; j <= 1; j++) {
            if (j == 0) {
                continue;
            }

            // 求得替换字符串 str
            int origin = pcs[i] - '0';
            // 形成一个 9 -> 0 的闭环
            int next = (origin + j) % 10;
            if (next == -1) {
                next = 9;
            }

            char[] clone = pcs.clone();
            // 将新的值重新赋值
            clone[i] = (char)(next + '0');
            String str = String.valueOf(clone);

            if (set.contains(str)) {
                continue;
            }
            if (cur.containsKey(str)) {
                continue;
            }

            // 如果在「另一方向」找到过，说明找到了最短路，否则加入队列
            if (other.containsKey(str)) {
                return step + 1 + other.get(str);
            } else {
                deque.addLast(str);
                cur.put(str, step + 1);
            }
        }
    }
    return -1;
}
```



## A* 算法

也是类似于BFS，但是在BFS的基础上，进行了相关的优化，对于无关的步骤，进行了简略

A*算法通过下面这个函数来计算每个节点的优先级:

![](http://image.tinx.top/20210625164829.png)

其中：

- f(n)是节点n的综合优先级。当我们选择下一个要遍历的节点时，我们总会选取综合优先级最高（值最小）的节点。
- g(n) 是节点n距离起点的代价。
- h(n)是节点n距离终点的预计代价，这也就是A*算法的启发函数。

f(n) 方法体为：

```java
static int f(String str) {
    int ans = 0;
    for (int i = 0; i < 4; i++) {
        int cur = str.charAt(i) - '0', target = t.charAt(i) - '0';
        int a = Math.min(cur, target), b = Math.max(cur, target);
        // 在「正向转」和「反向转」之间取 min
      	// b - a 是节点距离起点的代价
      	// a + 10 - b 是节点n距离重点的预计代价 
        int min = Math.min(b - a, a + 10 - b);
        ans += min;
    }
    return ans;
}
```

完整的A*算法描述如下：

```text
* 初始化open_set和close_set；
* 将起点加入open_set中，并设置优先级为0（优先级最高）；
* 如果open_set不为空，则从open_set中选取优先级最高的节点n：
    * 如果节点n为终点，则：
        * 从终点开始逐步追踪parent节点，一直达到起点；
        * 返回找到的结果路径，算法结束；
    * 如果节点n不是终点，则：
        * 将节点n从open_set中删除，并加入close_set中；
        * 遍历节点n所有的邻近节点：
            * 如果邻近节点m在close_set中，则：
                * 跳过，选取下一个邻近节点
            * 如果邻近节点m也不在open_set中，则：
                * 设置节点m的parent为节点n
                * 计算节点m的优先级
                * 将节点m加入open_set中
```



```java
static String s, t;
static Set<String> set = new HashSet<>();

static class Node {
    String str;
    int val, step;

    /**
     * str : 对应字符串
     * val : 估值（与目标字符串 target 的最小转换成本）
     * step: 对应字符串是经过多少步转换而来
     */
    Node(String _str, int _val, int _step) {
        str = _str;
        val = _val;
        step = _step;
    }
}

static int f(String str) {
    int ans = 0;
    for (int i = 0; i < 4; i++) {
        int cur = str.charAt(i) - '0', target = t.charAt(i) - '0';
        int a = Math.min(cur, target), b = Math.max(cur, target);
        // 在「正向转」和「反向转」之间取 min
        int min = Math.min(b - a, a + 10 - b);
        ans += min;
    }
    return ans;
}

public static int openLock(String[] deadends, String target) {
    s = "0000";
    t = target;
    if (s.equals(t)) {
        return 0;
    }

    for (String d : deadends) {
        set.add(d);
    }

    if(set.contains(target)){
        return -1;
    }

    PriorityQueue<Node> q = new PriorityQueue<>((a, b) -> a.val - b.val);
    Map<String, Node> map = new HashMap<>();
    Node root = new Node(s, f(s), 0);
    q.add(root);
    map.put(s, root);
    while (!q.isEmpty()) {
        Node poll = q.poll();
        char[] pcs = poll.str.toCharArray();
        int step = poll.step;
        if (poll.str.equals(t)) {
            return step;
        }

        for (int i = 0; i < 4; i++) {
            for (int j = -1; j <= 1; j++) {
                if (j == 0) {
                    continue;
                }
                int cur = pcs[i] - '0';
                int next = (cur + j) % 10;
                // 让 9 和 1 形成一个圆环
                if (next == -1) {
                    next = 9;
                }

                char[] clone = pcs.clone();
                clone[i] = (char) (next + '0');
                String str = String.valueOf(clone);

                if (set.contains(str)) {
                    continue;
                }
                // 如果 str 还没搜索过，或者 str 的「最短距离」被更新，则入队
                if (!map.containsKey(str) || map.get(str).step > step + 1) {
                    Node node = new Node(str, step + 1 + f(str), step + 1);
                    map.put(str, node);
                    q.add(node);
                }
            }
        }
    }
    return -1;
}
```

