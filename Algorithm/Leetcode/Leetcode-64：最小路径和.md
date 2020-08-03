## Leetcode-64：最小路径和

### 1. 动态规划

和题目62类似，我们定义`dp[i][j]`为从起点到位置`[i,j]`的最小路径和，那么我们可以得到公式`dp[i][j] = Math.min(dp[i-1][j], dp[i][j-1]) + grid[i][j]`，也就是可能从上面到达也可能从右边到达，那我们就选择最小的一个路径。

同样的我们可以将其优化为一维的`dp`数组。

```java
private int rows;
private int cols;
private int[] dp;
public int minPathSum(int[][] grid) {
    rows = grid.length;
    cols = grid[0].length;
    dp = new int[cols];
    dp[0] = grid[0][0];  // 开始的值
    for (int i = 1; i < cols; ++i) {   // 对于第一列，每个位置的值都是前面值的总和
        dp[i] = dp[i-1] + grid[0][i];
    }
    for (int i = 1; i < rows; ++i) {   // 对于每一行
        dp[0] = dp[0] + grid[i][0];    // 这里要注意一下，dp[0]代表上一层的第一列，那我们就需要更新为这一列的值
        for (int j = 1; j < cols; ++j) {
            dp[j] = Math.min(dp[j], dp[j-1]) + grid[i][j];  // 此时dp[j]代表上方过来的值，而dp[j-1]代表左侧过来的值
        }
    }
    return dp[cols-1];
}
```

