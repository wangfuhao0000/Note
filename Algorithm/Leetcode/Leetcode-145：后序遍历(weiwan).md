## Leetcode-145：后序遍历



```java
class Solution {
    public List<Integer> result = new ArrayList<>();
    public List<Integer> postorderTraversal(TreeNode root) {
        if(root == null) return result;
        TreeNode p = root;
        Stack<TreeNode> stack = new Stack<>();
        stack.push(p);  //根节点入栈
        while(!stack.isEmpty()) {
            TreeNode popNode = stack.peek();
            if(popNode == null) {
                stack.pop();
                result.add(stack.pop().val);
                continue;
            }
            stack.push(null);
            if(popNode.right != null) stack.push(popNode.right);
            if(popNode.left != null) stack.push(popNode.left);
        }
        return result;
    }
}
```

