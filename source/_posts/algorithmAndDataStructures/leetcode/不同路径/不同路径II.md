---
title: 不同路径II
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: 不同路径
---
### leetcode63 Unique Paths II

一个机器人位于一个 m x n 网格的左上角 （起始点在下图中标记为“Start” ）。

机器人每次只能向下或者向右移动一步。机器人试图达到网格的右下角（在下图中标记为“Finish”）。

问总共有多少条不同的路径？
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E4%B8%8D%E5%90%8C%E8%B7%AF%E5%BE%84/robot_maze.png)


例如，上图是一个7 x 3 的网格。有多少可能的路径？

说明：m 和 n 的值均不超过 100。

输入:
[
  [0,0,0],
  [0,1,0],
  [0,0,0]
]
输出: 2
解释:
3x3 网格的正中间有一个障碍物。
从左上角到右下角一共有 2 条不同的路径：
1. 向右 -> 向右 -> 向下 -> 向下
2. 向下 -> 向下 -> 向右 -> 向右


###
题解:
这道题跟上道题有些类似,也是使用动态规划来解决,但是要考虑更多的边界条件,因为有障碍物的存在,很多路径是不能走的.
和上题一样,我们先设dp[m,n]为m,n的不同路径结果.
根据上道题和测试过障碍物在只占一个格子是0种路径,


得出装填转移方程是:
$$
dp[m,n]
\begin{cases}
dp[i][j] = 0 , obstacleGrid[i][j] = 1(有障碍物的时候)\\
dp[i][j] = dp[i][j - 1] + dp[i - 1][j], obstacleGrid[i][j] = 0\\
\end{cases}
$$

```java
public static int uniquePaths(int m, int n) {
if (obstacleGrid.length > 100 || obstacleGrid[0].length > 100) {
        return 0;
    }

    int m = obstacleGrid.length;
    int n = obstacleGrid[0].length;
    int[][] dp = new int[m][n];

    for (int i = 0; i <= m - 1; i++) {
        for (int j = 0; j <= n - 1; j++) {
            if (i == 0 && j == 0) {
                dp[0][0] = obstacleGrid[0][0] == 1 ? 0 : 1;
            } else if (i == 0) {
                if (dp[i][j - 1] != 0 && obstacleGrid[i][j] != 1) {
                    dp[i][j] = 1;
                }
            } else if (j == 0) {

                if (dp[i - 1][j] != 0 && obstacleGrid[i][j] != 1) {
                    dp[i][j] = 1;
                }
            } else if (obstacleGrid[i][j] != 1) {
                dp[i][j] = dp[i][j - 1] + dp[i - 1][j];
            }
        }
    }
    return dp[m - 1][n - 1];
```
时间复杂度:o(mn)
空间复杂度:o(mn)
难点主要是在有障碍物的时候,要上一个点没有障碍物,当前的路径是通的才能算一条路经,就是代码中间的两种情况.
