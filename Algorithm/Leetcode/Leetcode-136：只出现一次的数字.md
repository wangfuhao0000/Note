## Leetcode-136：只出现一次的数字

### 1. 异或

就是利用亦或的特定，即**任何数字和自己异或都为0**。所以将所有的元素进行异或，那么出现奇数次的数字就会得以保留，而出现偶数次的数字会因为异或而变为0。

```java
public int singleNumber(int[] nums) {
    int result = 0;
    for (int num : nums) {
        result = result ^ num;
    }
    return result;
}
```

