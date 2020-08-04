## leetcode-739：每日温度

### 1. 暴力法

暴力法比较简单，就是双指针。维持一个指针来作为当前要求的元素，然后另一个指针从它的后一个元素向末尾遍历，直到遇到第一个比当前元素大的，最后两个下标一减就是当前元素的天数：

```java
class Solution {
    public int[] dailyTemperatures(int[] T) {
        int[] result = new int[T.length];
        for (int i = 0; i < T.length; ++i) {
            int j = i+1;  // j往后寻找第一个比T[i]大的
            for (; j < T.length && T[j] <= T[i]; ++j);
            if (j >= T.length)
                result[i] = 0;
            else {
                result[i] = j-i;
            }
        }
        return result;
    }
}
```

### 2. 单调栈

在用一个工具之前，一定是要分析一下这个问题的特点。首先这个问题和[leetcode-84](https://leetcode-cn.com/problems/largest-rectangle-in-histogram/)有点像，关键点在于求**第一个比自己小（大）的元素**。[leetcode-84](https://leetcode-cn.com/problems/largest-rectangle-in-histogram/)是找两边第一个比自己小的元素作为自己的左右边界，这样使用一个单调递增栈，那么**每个元素的左边界就已经确定了是栈里相邻的**，而有边界则只需要遇到的时候出栈就行。

这个题目也是一样的，**寻找第一个比自己大的**，那么我们就维持一个**单调递减**，始终让后面的比自己小。当遇到较大的说明需要出栈了，就依次将栈内比当前元素小的都出栈，这样就找到了每个元素遇到的第一个比自己大的。

```java
class Solution {
    public int[] dailyTemperatures(int[] T) {
        int[] result = new int[T.length];
        // 单调栈，存储的是下标
        Stack<Integer> stack = new Stack<>(); 
        for(int i = 0; i < T.length; ++i) {
            // 把所有栈内比自己小的都出栈，并计算到自己的距离
            while(!stack.isEmpty() && T[i] > T[stack.peek()]) {
                result[stack.peek()] = i-stack.pop();
            }
            stack.push(i);   // 找完符合条件的后，将自己入栈
        }
        // 将栈内找不到的全部相应置为0
        while(!stack.isEmpty())
            result[stack.pop()] = 0;
        return result;
    }
}
```

