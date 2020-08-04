## leetcode-230：二叉搜索树中第K小的元素

这个题目其实只要仔细思考就不算难，因为题目给定了是**二叉搜索树**，那么它的特点就是中序遍历是一个有序的数组。这样我们求第K小的元素时就可以利用中序遍历来递归完成。

首先我们维护一个变量`kValue`来记录我们**遍历过的节点数**。当我们遇到一个节点时先递归访问其左子树，然后访问根节点，访问根节点的时候就让`kValue--`，代表我们确实访问了一个元素，并判断`kValue`是否为0.如果为0，说明我们当前的元素恰好为第`k`个元素，直接记录下来就好，然后再递归地访问右子树。

```java
class Solution {
    private int result = 0;  // 记录结果
    private int kValue = 0;  // 记录访问地节点数
    public int kthSmallest(TreeNode root, int k) {
        this.kValue = k;
        helper(root);
        return result;
    }

    private void helper(TreeNode root) {
        if (root == null) return;
        // 遍历左子树
        helper(root.left);
        kValue--;  // 类似访问当前节点
        if (kValue == 0) {   // 当前恰好是第k个，记录结果并直接返回
            result = root.val;
            return;
        }
        // 遍历右子树
        helper(root.right);
    }
}
```

