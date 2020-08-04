## leetcode-124：二叉树中的最大路径和

这个题目的解法不是很好理解，首先我们引入一个**权重**的概念，也就是每个**子树**在计算的结果中所能占的权重。当我们对某个根节点`root`进行计算的时候，先计算得到左子树的权重，**且保证不能为负数**；然后计算右子树的权重，同样保证非负。然后将当前节点的值和左右子树的权重进行累加，就得到了当前子树的总和。进而拿这个和和最后结果进行比较，取最大值。比较完成后要注意，我们要返回左右子树中权重较大的那个值和当前根节点的和。

所以难理解的地方在于当我们计算时是`root.val+leftWeight+rightWeight`，但返回时却是`Math.max(leftWeight, rightWeight)+root.val`。这是因为当计算的时候我们假定**以当前节点作为根节点的树**能够有最大路径值，但返回的时候**我们的路径只可能取左子树或者右子树中的一个而不可能这条路径同时包含左右子树，那样就会重复经过当前的根节点了**，且如果取到某个子树那路径一定会经过根节点，所以返回的应该是`Math.max(leftWeight, rightWeight)+root.val`。

```java
class Solution {
    private int result = 0;
    public int maxPathSum(TreeNode root) {
        result = root.val;
        maxWeight(root);
        return result;
    }

    private int maxWeight(TreeNode root) {
        if (root == null) {
            return 0;
        }
        int leftWeight = Math.max(maxWeight(root.left), 0);  // 一定保证取到的是个非负值
        int rightWeight = Math.max(maxWeight(root.right), 0);
        int temp = root.val + leftWeight + rightWeight;  // 当前节点为根的整个树，看看能不能最大
        result = Math.max(result, temp);
        return Math.max(leftWeight, rightWeight) + root.val;  // 只能返回一个较大的路径上的值
    }
}
```

