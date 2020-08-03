## Leetcode-146：LRU Cache

这个题目就是使用的一个`HashMap`+双向链表。我们的思路是一步步的来优化，然后得到这样的一个结构。

首先我们需要能够存储数据，而根据这个缓存的特点需要能够记录一个最不常访问的节点，和最新访问过的节点。那我们选择使用一个双向链表，当访问到一个`Cache`时就将它放到链表的头部，这样每次访问一个就放在头部，那么链表末尾的自然存放的就是最不常被访问的。但只是使用双向链表有个缺点，就是根据`key`寻找对应`Cache`时，需要遍历整个链表。

于是我们就额外使用一个`Map`来存放一个`Key`和对应`Cache`的映射，那样我们就可以在`O(1)`复杂度下找到是否有对应的`Cache`了。

```java
class LRUCache {
    private class Node {
        int key;
        int val;
        Node pre;
        Node next;
    }

    private int capacity;
    private int size;
    private Node head;  // 存储头、尾节点
    private Node tail;
    private Map<Integer, Node> cache;

    public LRUCache(int capacity) {
        this.capacity = capacity;
        this.size = 0;
        head = new Node();
        tail = new Node();
        head.next = tail;  // 连接起来
        tail.pre = head;
        cache = new HashMap<>();
    }
    
    public int get(int key) {
        Node node = cache.get(key);
        if (node == null) {  // 如果缓存未命中，直接返回-1
            return -1;
        }
        // 先将节点从原来位置拿出来
        removeNode(node);
        // 插入到头节点
        insertHead(node);
        return node.val;
    }
    
    public void put(int key, int value) {
        Node node = cache.get(key);
        if (node != null) {  // 缓存已经存在了，替换value并把它放到头部
            node.val = value;
            removeNode(node);
            insertHead(node);
        } else {
          	// 缓存未命中需要放入，先看是否需要淘汰
            if (size == capacity) {   
                node = deleteTail();  // 淘汰尾部的
                cache.remove(node.key);   // 及时更新映射
                size--;
            }
            node = new Node();
            node.key = key;
            node.val = value;
            insertHead(node);   // 将新的缓存放到头部
            cache.put(key, node);  // 及时更新映射
            size++;
        }
    }

  	// 删除某个节点
    private void removeNode(Node node) {
        node.pre.next = node.next;
        node.next.pre = node.pre;
    }
		// 将节点插入到头部	
    private void insertHead(Node node) {
        node.pre = head;
        node.next = head.next;
        node.next.pre = node;
        node.pre.next = node;
    }
		// 淘汰尾部的节点
    private Node deleteTail() {
        Node node = tail.pre;
        node.pre.next = tail;
        node.next.pre = node.pre;
        node.next = null;
        node.pre = null;
        return node;
    }
}

/**
 * Your LRUCache object will be instantiated and called as such:
 * LRUCache obj = new LRUCache(capacity);
 * int param_1 = obj.get(key);
 * obj.put(key,value);
 */
```

