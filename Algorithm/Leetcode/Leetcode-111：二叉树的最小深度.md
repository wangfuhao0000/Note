## Leetcode-111：二叉树的最小深度

这个也是使用递归来解决的，但是和求最大深度是有差别的。求最大深度时，我们只需要求出左右子树中的最大值，然后加1并返回即可。但是求最小值就不一样了，如果还是按照类似的思路：求出左右子树中的最小值，然后加1并返回。那么假设当某个节点的右子树为`null`时，会导致它为0而直接返回，但其实左子树不为`null`，那么答案就是错误的了。

所以我们得到左右子树的最小深度时应该先进行一个判断，如果左右子树都不为`null`，那么就取两者的最小值；否则有一个不为`null`，那么就需要返回这个不为`null`的子树的最小深度。

```java
public int minDepth(TreeNode root) {
    if (root == null) {
        return 0;
    }
    int leftDepth = minDepth(root.left);
    int rightDepth = minDepth(root.right);
    if (leftDepth != 0 && rightDepth != 0) {   // 如果左右子树都不为null，则就返回最小值+1
        return Math.min(leftDepth, rightDepth) + 1;
    } else {   // 否则就需要返回那个不为null的子树深度+1
        return Math.max(leftDepth, rightDepth) + 1;
    }
}
```

