## leetcode-139：单词拆分

### 1. 递归法

递归法很容易理解，加入我当前的位置是start，那么我就往后end，直至字符串s[start : end]在字典里，那么我就将end作为新的start继续进行类似的遍历。但是当找到一个end的时候我们是不可以停止的，需要找到所有可能的end，并将其作为新的start进行遍历。可以在脑海里想象出这是一颗树的样子：

```java
class Solution {
    private List<String> wordDict;
    private String s;
    public boolean wordBreak(String s, List<String> wordDict) {
        this.wordDict = wordDict;
        this.s = s;
        return dfs(0);   // 从位置0开始
    }

    private boolean dfs(int start) {
        if (start == s.length()) return true;  // 说明前面的都恰好匹配完成，返回true
        int end = start+1;
        boolean temp = false;
        while (end <= s.length()) {
            if (contains(s.substring(start, end))) {
                temp = temp || dfs(end);  // 寻找每种可能，只要有一种返回了true说明可以遍历完成，直接返回true
                if (temp)
                    return true;
            }
            end++;
        }
        return temp;   // 说明没有能到头的
    }

    private boolean contains(String str) {
        for (String dict : wordDict) {
            if (dict.equals(str)) {
                return true;
            }
        }
        return false;
    }
}
```

当然这个方法是超时的，需要对其进行一定的优化。

### 2. 动态规划

其实能看出来我们第一种方法是有个弊端的，就是寻找每种可能时我们的end范围一直到了字符串末尾，其实只要到达字典中的最大长度即可。

动态规划的思路就是对于母串，我们只考虑从开始到当前位置组成的某个字串是否能由字段中的单词组成，并记为`dp[i]`。而对于区间`[0,i]`内我们将其进行划分，因为假设我们已经知道了dp[i]前面的结果，所以只需要判断`dp[j]`和`s.substr(j,i)`是否在字典中即可。如果`dp[j]`为`true`，说明区间`[0,j]`之间的字符串可以由字典内字符串组成，而区间`[j,i]`内的字符串如果也在字典里，说明区间`[0,i]`之间的字符串可以由字典内的字符串组成，因而可以停止判断，进而判断区间`[0,i+1]`内的字符串。

```java
public boolean wordBreak(String s, List<String> wordDict) {
    boolean[] dp = new boolean[s.length()+1];
    dp[0] = true;
    Set<String> dicts = new HashSet<>(wordDict);
    for (int i = 0; i <= s.length(); ++i) {
        for (int j = 0; j <= i; ++j) {  // j从0到i
            if (dp[j] && dicts.contains(s.substring(j, i))) {
                dp[i] = true;   // 区间[0,i]内的字符串可由字段内字符串组成，终止并进而判断[0,i+1]
                break;
            }
        }
    }
    return dp[s.length()];
}
```

