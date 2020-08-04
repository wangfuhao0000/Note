## leetcode-238：除自身以外数组的乘积

这个题目只要明白一点，除自身以外的乘积就是`左边的累积乘积×右边的累积乘积`。那么我们只需要构造`nums`的左累积乘积数组和右累积乘积数组即可。然后针对每个元素直接找到两个数组中对应的元素，相乘即可。

但题目有个要求，除了结果数组以外，空间复杂度为常数。所以我们要针对上面的进行优化。其实我们不需要右累积乘积数组，我们只需要维护一个变量`rightProduct`即可，当构造完左累积乘积数组后，我们从后往前遍历，边得到结果，边更改右累积乘积。

```java
class Solution {
    public int[] productExceptSelf(int[] nums) {
        int lens = nums.length;
        int[] result = new int[lens];
        result[0] = nums[0];
        int rightProduct = 1;
        // 先初始化result数组, 得到左边的累积数组，注意最后一个位置不要构造
        for (int i = 1; i < lens-1; ++i) {
            result[i] = result[i-1] * nums[i];
        }
        for (int i = lens-1; i > 0; --i) {
            result[i] = result[i-1] * rightProduct;
            rightProduct *= nums[i];   // 边得到结果，边更改右累积乘积
        }
        result[0] = rightProduct;   // 最后一个元素就是得到的前面所有元素的右累积乘积
        return result;
    }
}
```

