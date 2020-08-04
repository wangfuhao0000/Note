## leetcode-84：柱状图中的最大矩形

### 1. 中心扩展法

和求回文字符串一个思想，就是根据当前`pos`的矩形向左右两方进行扩展，扩展的条件就是左右的矩形的高度不能比当前`pos`的矩形低，否则就不能构造成矩形了，这样矩形的高度就是当前`pos`矩形的高度。扩展完了后得到左右边界，也就是得到了矩形的宽。这样求得形成的矩形的高度和宽度后，得到矩形的面积，最后取一个最大值。

```java
class Solution {
    private int[] heights;
    private int result = 0;
    public int largestRectangleArea(int[] heights) {
        this.heights = heights;
        for (int i = 0; i < heights.length; ++i) {
            span(i);   // 以每个矩形为中心进行扩展
        }
        return result;
    }

    private void span(int pos) {
        int left = pos-1, right = pos+1;
        int curHeight = heights[pos];
        while (left >= 0 && heights[left] >= curHeight) {
            left--;
        }
        left++;   // 左边界
        while (right < heights.length && heights[right] >= curHeight) {
            right++;
        }
        right--;   // 右边界 
        result = Math.max(result, (right-left+1)*curHeight);   // 取一个较大值
    }
}
```

### 2. 单调栈

这个是根据第一种方法来改进的，通过第一种我们知道了其实就是求左边和右边第一个比自己小的元素，我们暂时叫它们左右边界。那么我们就维护一个**单调递增栈**，通过这个栈我们能**立即知道当前元素的左边界是谁**，这样在我们遇到一个右边界时，就将栈内所有符合条件（肯定知道了左边界，那么就是**以当前元素作为右边界的值**）出栈。这样就能求得对应的高（出栈的值）和宽（当前元素下标和出战后栈顶的下标之差）。

```java
class Solution {
    public int largestRectangleArea(int[] heights) {
        Stack<Integer> stack = new Stack<>();   // 单调递增栈
        stack.push(-1);
        int result = 0;
        for (int i = 0; i < heights.length; ++i) {
            // 把比自己大的都出栈
            while (stack.peek() != -1 && heights[i] < heights[stack.peek()]) {  
                int curHeight = heights[stack.pop()];
                int curWidth = i - stack.peek() - 1;
                result = Math.max(result, curHeight * curWidth);
            }
            stack.push(i);
        }
        // 假如就是个单调递增的，那么栈里会都保存，所以进行同样的处理
        while(stack.peek() != -1) {
            int curPos = stack.pop();
            int curHeight = heights[curPos];
            int curWidth = heights.length - stack.peek() - 1;  // 此时右边界就是heights.length
            result = Math.max(result, curHeight*curWidth);
        }
        return result;
    }
}
```

