## leetcode-3：无重复字符的最长子串

### 1. 暴力法

思路很简单，就是当我们遍历一个字符的时候，看看它在原来的位置出现过没有，加入当前的字符位置为`pos1`，而往前遍历时发现它在`pos2`中出现过，那么所得到的不重复的字符长度就是`pos1-pos2`。

但这个过程有一个问题，加入我们判断的字符串为`abcba`，那么当我们遍历到第二个字符`b`时`pos1`为3，往前查找可以得到`pos2`为1，那么最大长度就是`pos1-pos2 = 2`。继续此过程当遍历到第二个字符`a`时`pos1`为4，往前查找可得到`pos2`为0，那么得到的最大长度就是`pos1-pos2 = 4`，但其实我们忽略了字符`b`这个信息。

所以我们要维持一个左边界`leftSide`，它始终指向不重复字串的开始处，每次遇到重复字符时，就更新这个左边界，让它变为重复字符后面的一个位置。那样就能保证左边界到当前字符的前一个字符之间没有重复字符。

![image-20200624203045913](E:%5C%E7%AC%94%E8%AE%B0%5C%E5%9B%BE%E7%89%87%5Cimage-20200624203045913.png)

```java
public int lengthOfLongestSubstring(String s) {
    if ("".equals(s))
        return 0;
    int leftSide = 0;
    int len = 1;  // 最少有一个不重复的
    for (int i = 1; i < s.length(); ++i) {
        char cur = s.charAt(i);
        int j = i-1;  // 向前遍历，直到找到重复的，或者左边界
        while (j >= leftSide && s.charAt(j) != cur)
            --j;
        len = Math.max(len, i-j);   // 更新一下长度
        leftSide = Math.max(leftSide, j+1);   // 更新左边界，注意这里时j+1，因为j指向的可能是重复的
    }
    return len;
}
```

### 2. 暴力法优化

其实这个优化就是针对于向前遍历的过程，当我们遍历完一个字符时将此字符和对应的位置存储到一个哈希表中，这样我们要找时直接从哈希表中查找就行了。其实这种思路反而更好理解一些，当我从map中找当前字符时，如果找到那我就无脑的将`lsefSide`设置为它后面的位置，这样才能保证不重复。

```java
public int lengthOfLongestSubstring(String s) {
    if ("".equals(s))
        return 0;
    Map<Character, Integer> map = new HashMap<>();
    map.put(s.charAt(0), 0);
    int leftSide = 0;
    int len = 1;
    for (int i = 1; i < s.length(); ++i) {
        char cur = s.charAt(i);
        if (map.containsKey(cur)) {  // 如果遍历过这个字符，则需要把leftSide设置为它后面的位置，保证不重复
            leftSide = Math.max(leftSide, map.get(cur)+1);
        }
        map.put(cur, i);  //把当前字符放到map中，它可能是新放入，也可能是更新原有值，但位置总是变大
        len = Math.max(len, i-leftSide+1);  //因为leftSide指向是不重复的，所以需要+1
    }
    return len;
}
```



