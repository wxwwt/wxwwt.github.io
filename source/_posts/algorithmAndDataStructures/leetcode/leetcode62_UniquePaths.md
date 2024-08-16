---
title: leetcode62_UniquePaths
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: leetcode
---
### leetcode62 Unique Paths

一个机器人位于一个 m x n 网格的左上角 （起始点在下图中标记为“Start” ）。

机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为“Finish”）。

问总共有多少条不同的路径？
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E4%B8%8D%E5%90%8C%E8%B7%AF%E5%BE%84/robot_maze.png)


例如，上图是一个7 x 3 的网格。有多少可能的路径？

说明：m 和 n 的值均不超过 100。

示例 1:

输入: m = 3, n = 2
输出: 3
解释:
从左上角开始，总共有 3 条路径可以到达右下角。
1. 向右 -> 向右 -> 向下
2. 向右 -> 向下 -> 向右
3. 向下 -> 向右 -> 向右
示例 2:

输入: m = 7, n = 3
输出: 28

###
题解:
这道题使用动态规划是比较好解决的,我们可以先设dp[m,n]为m,n的不同路径结果.
我们根据题目分析后得出
dp[1,1]是1
dp[1,2]也是1
dp[2,1]也是1

dp[2,2]往右边走的话其实就是dp[1,2],往下面走的话就是dp[2,1],所以dp[2,2] = dp[1,2] + dp[2,1].
dp[2,3]也可以根据往右边走等于dp[1,3],往下走其实就是dp[2,2]的结果,所以dp[2,3] = dp[1,3] + dp[2,2].
我们推出dp[m,n] = dp[m - 1][n] + dp[m][n - 1].
得出装填转移方程是:
$$
dp[m,n]
\begin{cases}
1, &if m == 1 || n == 1\\
dp[m,n] = dp[m - 1][n] + dp[m][n - 1]\\
\end{cases}
$$

```java
public static int uniquePaths(int m, int n) {
       if (m > 100 || n > 100) {
           return 0;
       }

       int[][] dp = new int[m + 1][n + 1];
       for (int i = 1; i <= m; i++) {
           for (int j = 1; j <= n; j++) {
               if (i == 1 || j == 1) {
                   dp[i][j] = 1;
               } else {
                   dp[i][j] = dp[i - 1][j] + dp[i][j - 1];
               }
           }
       }
       return dp[m][n];
   }
```
时间复杂度:o(mn)
空间复杂度:o(mn)

因为最后的结果其实只依赖于前面两个 dp[i - 1][j] + dp[i][j - 1]的结果,所以我们可以把二维数组简化掉.
```java
if (m > 100 || n > 100) {
           return 0;
       }

      int[] dp = new int[m > n ? m : n];
       Arrays.fill(dp, 1);
       for (int i = 1; i < m; i++) {
           for (int j = 1; j < n; j++) {
                  dp[j] = dp[j] + dp[j - 1];
           }
       }
       return dp[n -1];
```
时间复杂度:o(mn)
空间复杂度:o(max(n,m))
