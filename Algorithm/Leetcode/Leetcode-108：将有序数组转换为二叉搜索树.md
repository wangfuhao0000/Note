## Leetcode-108：将有序数组转换为二叉搜索树

这个题目就使用递归就可以了，因为是个排序数组，且要求是平衡的。假设我们这个数组的开始和结束下标是`left`和`right`，我们可以这样构造此树：

- 首先找到当前数组的中点`mid`，将此中点作为当前树的根节点，这样这棵树的左子树和右子树长度就相等或者相差1，保证了平衡性。
- 构造完当前节点后，递归的构造左子树和右子树。而左子树的开始下标和结束下标为`left`和`mid-1`；右子树的开始下标和结束下标为`mid+1`和`right`。

同样由于是个有序数组，这也保证了构造出来的二叉树是一个排序二叉树。

```java
class Solution {
    private int[] nums;
    public TreeNode sortedArrayToBST(int[] nums) {
        this.nums = nums;
        return init(0, nums.length-1);
    }

    private TreeNode init(int left, int right) {
        if (left > right) {
            return null;
        } else if (left == right) {  // 只有一个节点了
            return new TreeNode(nums[left]);
        }
        int mid = left + (right-left) / 2;
        TreeNode root = new TreeNode(nums[mid]);  // 构造当前树的根节点，并递归构造左右子树
        root.left = init(left, mid-1);
        root.right = init(mid+1, right);
        return root;
    }
}
```

