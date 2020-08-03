## Leetcode-148：排序链表

### 1. 插入排序

这个方法的思路就是插入排序，即先创建一个新的链表存储最后的结果，然后不断的从原来的链表中取节点，并放到结果链表中的适当位置。而寻找这个位置的过程类似于插入排序，即从链表头开始寻找比它大的第一个元素，然后放在这个元素前面就可以了。

```java
public ListNode sortList(ListNode head) {
    if (head == null || head.next == null)
        return head; 
    ListNode result = new ListNode(-1);
    ListNode p = head, nxt = p.next;  //nxt用来存储要取的节点的下一个
    while (p != null) {
        nxt = p.next;   // 存储下一个
        ListNode pos = result;  // 从头开始寻找要插入的位置
        while (pos.next != null && pos.next.val <= p.val) {
            pos = pos.next;
        }
        //然后放在pos的后面
        p.next = pos.next;
        pos.next = p;
      	// 获得新的节点
        p = nxt;
    }
    return result.next;
}
```

### 2. 归并排序

这种方法也比较好理解，就是先利用快慢指针将链表分成两部分，对左右两部分的链表分别进行排序，排序后将两个有序链表合并即可。

```java
class Solution {
    public ListNode sortList(ListNode head) {
        if (head == null || head.next == null) {
            return head;
        }
        ListNode slow = head, fast = slow.next;
        while (fast != null && fast.next != null) {
            slow = slow.next;
            fast = fast.next.next;
        }
      	// 使用快慢指针找到链表中点
        ListNode rightHead = slow.next;
        slow.next = null;   // 把链表从中间断开
        ListNode l1 = sortList(head);   // 分别排序左右两部分
        ListNode l2 = sortList(rightHead);
        head = merge2list(l1, l2);   // 排序完后进行合并并返回
        return head;
    }

    private ListNode merge2list(ListNode l1, ListNode l2) {
        ListNode result = new ListNode(-1), p3 = result;
        ListNode p1 = l1, p2 = l2;
        while (p1 != null && p2 != null) {
            if (p1.val <= p2.val) {
                p3.next = new ListNode(p1.val);
                p1 = p1.next;
            } else {
                p3.next = new ListNode(p2.val);
                p2 = p2.next;
            }
            p3 = p3.next;
        }
        if (p1 != null) {
            p3.next = p1;
        } else {
            p3.next = p2;
        }
        return result.next;
    }
}
```

