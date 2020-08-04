## leetcode-96：不同的二叉搜索树

这个题目的关键就是DP公式的推导，dp数组中的每一项`dp[n]`代表`n`个节点能构造成的二叉搜索树的个数。

```java
class Solution {
    public int numTrees(int n) {
        int[] G = new int[n+1];
        G[0] = G[1] = 1;
        for (int i = 2; i < n+1; i++) {     // 对于节点数为i
            for (int j = 0; j < i; j++) {   // 将当前节点作为根节点，左右节点
                G[i] += G[j] * G[i-j-1];
            }
        }
        return G[n];
    }
}
```

