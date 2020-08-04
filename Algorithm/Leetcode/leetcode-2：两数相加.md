## leetcode-2：两数相加

### 1. 按照相加规则

就按照普通的加法规则将两个数字相加即可，其中要保存一个变量就是每次相加后的进位`carry`。而且可能两个数字的位数不一样多，所以当某一个链表遍历到头时，我们将代表它的值设置为0，这样可以一直遍历直到两个链表都为`null`为止。

```java
public ListNode addTwoNumbers(ListNode l1, ListNode l2) {
    int carry = 0;
    ListNode result = new ListNode(-1);  //保存最后的结果
    ListNode p1 = l1, p2 = l2, p3 = result;
    while (p1 != null || p2 != null) {
        int val1 = p1 == null ? 0 : p1.val;  //谁为空，则设置相应值为0
        int val2 = p2 == null ? 0 : p2.val;
        int sum = val1 + val2 + carry;   //别忘记加进位carry
        p3.next = new ListNode(sum % 10);
        carry = sum / 10;    //更新一下进位信息
        p1 = p1 == null ? null : p1.next;
        p2 = p2 == null ? null : p2.next;
        p3 = p3.next;
    }
    if (carry != 0) {
        p3.next = new ListNode(carry);   //要考虑到最后的进位
    }
    return result.next;
}
```

