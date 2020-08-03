## Leetcode-138：复制带随机指针的链表

### 1. 深度优先遍历

即使维护一个`Map`，里面存放的就是旧的节点和新的节点之间的映射。当遇到一个新节点时如果此节点还没有复制过，则先复制这个节点。然后递归地去复制它的`next`和`random`。

```java
class Solution {
    private Map<Node, Node> map = new HashMap<>();
    public Node copyRandomList(Node head) {
        if (head == null) {
            return head;
        }
        if (map.containsKey(head)) {   // 如果这个节点已经复制过了，直接返回
            return map.get(head);
        }
        Node node = new Node(head.val);   // 否则复制这个节点，并放到Map中
        map.put(head, node);
        node.next = copyRandomList(head.next);   // 递归复制它的next和random
        node.random = copyRandomList(head.random);
        return node;
    }
}
```

### 2. 在节点后复制节点

