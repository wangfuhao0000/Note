## Leetcode-103：二叉树的锯齿形层次遍历

和层次遍历类似，但要注意的是在每一层遍历完成后查看这一层是基数层还是偶数层，如果是偶数层则需要将集合里的元素进行一个反转操作。

```java
public List<List<Integer>> zigzagLevelOrder(TreeNode root) {
    List<List<Integer>> result = new ArrayList<>();
    if (root == null) {
        return result;
    }
    Queue<TreeNode> queue = new LinkedList<>();
    queue.offer(root);
    while (!queue.isEmpty()) {
        List<Integer> temp = new ArrayList<>();
        for (int i = queue.size(); i > 0; --i) {  // 一定是从最大到0，可保证从队列取出的元素是上一层的
            TreeNode popNode = queue.poll();
            temp.add(popNode.val);
            if (popNode.left != null) {
                queue.offer(popNode.left);
            }
            if (popNode.right != null) {
                queue.offer(popNode.right);
            }
        }
        if (result.size() % 2 == 1) {  // 说明已经遍历了奇数层，当前是偶数
            Collections.reverse(temp);
        }
        result.add(temp);
    }
    return result;
}
```

