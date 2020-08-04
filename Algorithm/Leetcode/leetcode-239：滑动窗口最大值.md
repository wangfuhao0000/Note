## leetcode-239：滑动窗口最大值

### 双向队列

这个题目使用**双端的单调队列**来做。我们维持一个双端队列，即队列的两端都可以添加和删除元素，而且队列里面的元素保持递减，即队列的开头是最大的，队尾是最小的。**这样队列里的最大值始终是队头元素**。

- 为了维护这个队列的单调性，当在队尾添加元素时，**把比当前元素小的都出队列**，然后再添加。

- 当要取得最大值的时候，直接取队头的元素即可

- 当窗口滑动时，将刚才窗口的左侧元素从队列中去除掉。如果它是最大值，那么就是将队列的头部去掉，否则不会影响队列中的元素。

```java
public class solution_239 {

    class Dequeue {
        ArrayDeque<Integer> queue;

        public Dequeue() {
            queue = new ArrayDeque<>();
        }

        public void add(int num) {
            // 添加时把比当前元素小的都清除掉
            while (!queue.isEmpty() && queue.peekLast() < num)
                queue.pollLast();
            queue.addLast(num);  // 添加到队尾
        }

        public int max() {
            return queue.peekFirst();   // 因为单调，最大值一定是队头
        }

        public void remove(int num) {
            // 当窗口左侧的值是最大值时，将队列中当前的头部去掉
            if (!queue.isEmpty() && num == queue.peekFirst())  
                queue.pollFirst();
        }
    }

    public int[] maxSlidingWindow(int[] nums, int k) {
        int[] result = new int[nums.length-k+1];
        int index = 0;
        Dequeue dequeue = new Dequeue();
        for (int i = 0; i < nums.length; ++i) {
            if (i < k-1) {  // 先将前面k-1个元素加入到队列中
                dequeue.add(nums[i]);
            } else {
                dequeue.add(nums[i]);   // 此时肯定有了k个元素，然后取出最大值
                result[index++] = dequeue.max();
                dequeue.remove(nums[i-k+1]);   // 窗口滑动，将刚才窗口左侧的元素尝试从队列中移除（最大值时才去除）
            }
        }
        return result;
    }

}
```

### 优先队列

其实也可以使用一个长度为`k`的优先队列（大顶堆），窗口滑动时将新添加的元素加到优先级队列中，将划过的元素从队列中删除。并取得当前窗口的最大值为堆顶元素。

```java
class Solution {
    public int[] maxSlidingWindow(int[] nums, int k) {
        PriorityQueue<Integer> queue = new PriorityQueue<>(k, new Comparator<Integer>() {
            @Override
            public int compare(Integer o1, Integer o2) {
                return o2-o1;
            }
        });
        for(int i = 0; i < k; ++i)
            queue.offer(nums[i]);
        int[] result = new int[nums.length-k+1];
        result[0] = queue.peek();
        for(int i = k; i < nums.length; ++i) {
            queue.remove(nums[i-k]);   // 删除划过的元素
            queue.offer(nums[i]);      // 将新的元素添加
            result[i-k+1] = queue.peek();   // 最大元素为堆顶
        }
        return result;
    }
}
```

