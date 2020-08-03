## leetcode-56：合并区间

### 1. 双指针

这个方法我们就使用**双指针**。

1. 首先我们将这些区间按照区间的开始进行合并，这样每个区间的开始就会从小到大进行排列
2. 然后我们维持两个指针`pre`和`next`。其中：`pre`指向的是当前可合并的最后一个区间，`next`则从`pre`的后面往后遍历，查找能和`pre`进行合并的区间。
3. 如果找到了，则将`next`所指的区间合并到`pre`中；否则说明`pre`不能和后面的区间合并了，将`pre`变为现在的`next`，此处要注意`pre`一定是变为`next`而不是加1，因为这两者之间的区间已经合并了。

当然实际编程的时候有一些注意的点，例如：

- 题目给的是数组，但我们最好是自己定义一个区间类，来存储每个区间
- 还要注意当合并区间时，合并完了一个`pre`后，就将它加入到一个新的结果集中，而且在合并时不能删除集合中的元素，否则下标会因此变乱。

```java
class Solution {
    private class Interval {  //定义一个区间类
        int left;
        int right;
        public Interval(int left, int right) {
            this.left = left;
            this.right = right;
        }
    }
    public int[][] merge(int[][] intervals) {
        List<Interval> list = new ArrayList<>(intervals.length);
        for (int[] interval : intervals) {
            Interval t = new Interval(interval[0], interval[1]);
            list.add(t);
        }
        // 根据区间的开始进行排序
        Collections.sort(list, (a,b) -> {
            return a.left - b.left;
        });
      
        int pre = 0, next = pre + 1;
        List<Interval> temp = new ArrayList<>();  // 存储最后的结果
        while (pre < list.size()) {
            next = pre + 1;  // next始终从pre的后一个位置向后遍历
            while (next < list.size() && list.get(next).left<=list.get(pre).right) {  //如果能合并，则进行合并
                list.get(pre).right = Math.max(list.get(next++).right, list.get(pre).right);  
            }
            temp.add(list.get(pre));   //合并完此时pre肯定不能继续和后面的合并了，将其加入到结果集中
            pre = next;    // 更改pre为next
        }
      
      	// 将结果转为数组
        int[][] result = new int[temp.size()][2];
        for (int i = 0; i < temp.size(); ++i) {
            result[i][0] = temp.get(i).left;
            result[i][1] = temp.get(i).right;
        }
        return result;
    }
}
```

