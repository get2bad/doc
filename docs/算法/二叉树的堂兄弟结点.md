# [二叉树的堂兄弟节点](https://leetcode-cn.com/problems/cousins-in-binary-tree/)

在二叉树中，根节点位于深度 0 处，每个深度为 k 的节点的子节点位于深度 k+1 处。

如果二叉树的两个节点深度相同，但 父节点不同 ，则它们是一对堂兄弟节点。

我们给出了具有唯一值的二叉树的根节点 root ，以及树中两个不同节点的值 x 和 y 。

只有与值 x 和 y 对应的节点是堂兄弟节点时，才返回 true 。否则，返回 false。

 

示例 1：

![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/02/16/q1248-01.png)

输入：root = [1,2,3,4], x = 4, y = 3
输出：false
示例 2：

![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/02/16/q1248-02.png)

输入：root = [1,2,3,null,4,null,5], x = 5, y = 4
输出：true
示例 3：

![](https://assets.leetcode-cn.com/aliyun-lc-upload/uploads/2019/02/16/q1248-03.png)

输入：root = [1,2,3,null,4], x = 2, y = 3
输出：false



遇到二叉树问题，就一定会涉及到遍历的问题，涉及到遍历问题，就会有以下：

+ DFS 深度优先搜索遍历

  一般使用递归方式来找出深度

+ BFS 广度优先搜索遍历

  一般使用数据结构来找出广度

因为我们是来求相同深度的结点的父节点是否不一致，那么我们可以考虑使用递归方式的```深度优先搜索遍历```

1. 首先我们要先求目标结点的深度

   > 我们使用递归的方式，求目标值的深度信息
   >
   > 本方法的思想就是： 如果找不到就继续调用这个方法，深度值+1，如果真的找不到就返回 -1

```java
// 获取深度
private static int getDeepth(TreeNode tree, int target,int deepth){
    if(tree != null){
        if(tree.val == target) return deepth;
        int l = getDeepth(tree.left,target,deepth + 1);
        int r = getDeepth(tree.right,target,deepth + 1);
        // 如果左边没有事-1就返回右边的r，否则就返回左子树的深度
        return l == -1? r: l;
    }
    return -1;
}
```

2. 获取完毕深度以后，我们要判断这两个值的父节点是否是一个结点

   > 本方法依旧是采用递归的方式，如果发现不是同一个父节点，就直接返回true

```java
// 获取是否为同一父节点
public static boolean isFatherNode(TreeNode tree,int x,int y){
    if(tree != null){
        if(tree.left != null && tree.right != null){
            // 判断如果是同一个父节点，那么就直接返回false
            if((tree.left.val == x && tree.right.val == y) || (tree.left.val == y && 		tree.right.val == x)){
                return false;
            }
        }
        return isFatherNode(tree.left,x,y) && isFatherNode(tree.right,x,y);
    }
    return true;
}
```

3. 整合方法

```java
public static boolean isCousins(TreeNode root, int x, int y) {
    return getDeepth(root,x,1) == getDeepth(root,y,1) && isFatherNode(root, x, y);
}
```

