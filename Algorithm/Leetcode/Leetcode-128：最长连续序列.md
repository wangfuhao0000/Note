## Leetcode-128:最长连续序列

### 1. 使用set

其实就是先将所有元素放到一个`Set`里，然后针对于每个元素，看看以它为中心向两边扩展，看最多能扩展多长。扩展的时候就可以利用`Set`的机制查找是否存在，如果存在顺便删除元素，避免重复判断。

```java
public int longestConsecutive(int[] nums) {
    Set<Integer> set = new HashSet<>();
    for(int num : nums)
        set.add(num);
    int result = 0;
    for(int num : nums) {
        int cur = 1;  // 以当前为中心向两边扩展
        for(int i = num-1; set.contains(i); --i) {
            cur++;
            set.remove(i);
        }
        for(int i = num+1;set.contains(i); ++i) {
            cur++;
            set.remove(i);
        }
      	// 扩展完成后得到新的结果
        result = Math.max(result, cur);
    }
    return result;
}
```

