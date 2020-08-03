## Leetcode-109：将有序链表转换为二叉搜索树

和108题类似，只是有两个需要注意的地方：

- 查找中间节点，因为无法直接访问，所以需要快慢指针来找到链表的中点
- 构造左右子树时因为需要将刚刚的根节点去掉，所以需要将找到的中间节点从链表中断开，前后都要断开。

```java
class Solution {
    public TreeNode sortedListToBST(ListNode head) {
        return helper(head);
    }
    
    private TreeNode helper(ListNode head) {
        if (head == null)
            return null;
        if (head.next == null)
            return new TreeNode(head.val);
        ListNode p_slow = head, p_fast = head, pre = null;  // pre就是为了将中间节点与前面的链表断开
        while(p_fast != null && p_fast.next != null) {
            pre = p_slow;
            p_slow = p_slow.next;
            p_fast = p_fast.next;
            if (p_fast != null)
                p_fast = p_fast.next;
            else
                break;
        }
        TreeNode root = new TreeNode(p_slow.val);  // p_slow是中间节点，构造为当前的根节点
        ListNode rightStart = p_slow.next;  // 右子树开始
      	// 将根节点从原来链表中断开
        p_slow.next = null;
        pre.next = null;
      	// 递归构造左子树和右子树
        root.left = helper(head);
        root.right = helper(rightStart);
        return root;
    }
}
```

