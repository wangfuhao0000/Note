## leetcode-236：二叉树的最近公共祖先

这个题目是使用递归来做，我们找祖先其实就是找一个节点，它的左右子树包含要找的两个节点。

当我们遍历一个节点`root`时：

- 如果它是`p`或者`q`那我们就直接返回当前节点。
- 否则我们就去左子树和右子树去找，根据找到的结果来返回不同的值：
  - 如果左右子树都找到了，说明当前节点就是最近的公共祖先，直接返回
  - 如果只找到了一个，我们就只返回找到的那个。（说明**另一个子树上没有`p`和`q`，则最近的公共祖先就是找到的那个**。）
  - 如果都没找到，说明当前节点所在的子树没有那两个节点，返回`null`。

```java
class Solution {
    private TreeNode p;
    private TreeNode q;
    public TreeNode lowestCommonAncestor(TreeNode root, TreeNode p, TreeNode q) {
        this.p = p;
        this.q = q;
        return helper(root);
    }

    private TreeNode helper(TreeNode root) {
        if (root == null) return root;

        if (root == p || root == q) {  // 找到了其中一个
            return root;
        }
        TreeNode leftNode = helper(root.left);  // 去左边找
        TreeNode rightNode = helper(root.right);   // 去右边找

        if (leftNode != null && rightNode != null) {  // 左右都能找到，则结果就是自己
            return root;
        } else if (leftNode != null || rightNode != null) {   // 只找到了一个，就返回找到的那个
            return leftNode==null ? rightNode : leftNode;  
        } else {
            return null;
        }
    }
}
```

