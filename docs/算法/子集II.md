# [子集 II](https://leetcode-cn.com/problems/subsets-ii/)

给你一个整数数组 `nums` ，其中可能包含重复元素，请你返回该数组所有可能的子集（幂集）。

解集 **不能** 包含重复的子集。返回的解集中，子集可以按 **任意顺序** 排列。

 

**示例 1：**

```
输入：nums = [1,2,2]
输出：[[],[1],[1,2],[1,2,2],[2],[2,2]]
```

**示例 2：**

```
输入：nums = [0]
输出：[[],[0]]
```



本题直接使用 双层遍历即可解决问题，最关键就是**for循环的条件，还有添加的条件**

```java
// 穷举法 + 双层遍历
public static List<List<Integer>> subsetsWithDup(int[] nums) {
    Arrays.sort(nums);
    int n = nums.length;
    Set<List<Integer>> ans = new HashSet<>();
    List<Integer> cur = new ArrayList<>();

    // 枚举 i 代表，枚举所有的选择方案状态
    // 例如 [1,2]，我们有 []、[1]、[2]、[1,2] 几种方案，分别对应了 00、10、01、11 几种状态
    for (int i = 0; i < (1 << n); i++) {
        cur.clear();
        // 对当前状态进行诸位检查，如果当前状态为 1 代表被选择，加入当前方案中
        for (int j = 0; j < n; j++) {
            int t = (i >> j) & 1;
            if (t == 1) cur.add(nums[j]);
        }
        // 将当前方案中加入结果集
        ans.add(new ArrayList<>(cur));
    }
    return new ArrayList<>(ans);
}
```