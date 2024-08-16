---
title: LeetCode-396周赛记录
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: commonStructures
---
LeetCode-396周赛记录

# Q1：

https://leetcode.cn/problems/find-the-k-or-of-an-array/description/

这个第一题倒是不复杂，但是问题是这个文字描述太绕了，根本就没搞清楚它是要干嘛。弄清楚题意之后，翻译成将nums中的每个元素里面找出二进制位数是1的，如果某一位上1的数量达到了k，就记录下来，最后求得值就是这几位为1的元素，转为二进制的和。比如示例一给的是，第0，3位是满足了有k个1，就计算2^0 + 2^3 = 9。后面的示例二和示例三可以忽略，简直是用来误导人的，让你搞不明白到底是要算&还是|。因为nums[i] < 2^31所以遍历到30位就行了。

位运算的操作可以看下灵神的文章：[分享｜从集合论到位运算，常见位运算技巧分类总结！](https://leetcode.cn/circle/discuss/CaOJ45/)

```
 public int findKOr(int[] nums, int k) {
        int sum = 0;
        int[] count = new int[31];
        for (int i = 0; i < 31; i++) {
            for (int j = 0; j < nums.length; j++) {
                count[i] += (nums[j] >> i) & 1;
                if (count[i] >= k) {
                    sum = sum | (1 << i);
                }
            }
        }
        return sum;
    }
```





# Q2：

第二题纯计算出来的，就是如果sum1和sum2比较，哪个小就得往小的上加数值，让两边数值一样大，如果小的那个数里面没有0，那就没法变大，就要跳出循环返回-1。

```
   public long minSum(int[] nums1, int[] nums2) {

        long countZ1 = 0, countZ2 = 0, sum1 = 0, sum2 = 0;
        for (int i = 0; i < nums1.length; i++) {
            if (nums1[i] == 0) {
                countZ1++;
            } else {
                sum1 += nums1[i];
            }

        }

        for (int i = 0; i < nums2.length; i++) {
            if (nums2[i] == 0) {
                countZ2++;
            } else {
                sum2 += nums2[i];
            }
        }


        if (sum1 + countZ1 < sum2 + countZ2) {
            while (sum1 + countZ1 < sum2 + countZ2) {
                if (countZ1 > 0) {
                    countZ1 = sum2 + countZ2 - sum1;
                } else {
                    break;
                }

            }
        } else if (sum1 + countZ1 == sum2 + countZ2) {
            return sum1 + countZ1;
        } else {
            while (sum1 + countZ1 > sum2 + countZ2) {
                if (countZ2 > 0) {
                    countZ2 = sum1 + countZ1 - sum2;
                } else {
                    break;
                }
            }
        }

        return sum1 + countZ1 == sum2 + countZ2 ? sum1 + countZ1 : -1;
    }
```





# Q3：

[100107. 使数组变美的最小增量运算数](https://leetcode.cn/problems/minimum-increment-operations-to-make-array-beautiful/)

第三题一开始没写出来，赛后看了别人的题解才弄明白。

我们可以这样想，只要每三个数里面有一个数字是等于k的就满足题意了，所以题目就转变成加上的数值最小可以保证每三个数里面有一个k。

思路这里有两种

思路一：从最后一个元素开始想，如果最后一个元素增加，那么它前面两个元素就可以不增加。

如果最后一个元素不增加，那么前面两个元素会有其中一个增加。

用dp(i,j)来表示第i元素右边有几个元素没有增加的状态就可以得到：

dp(i,j) = dp(i-1,2) + max(k - nums[i], 0);

dp(i,j) = dp(i-1,j+1)， j < 2;

```
  public long minIncrementOperations(int[] nums, int k) {
        int n = nums.length;
        long[][] memo = new long[n][3];
        for (long[] m : memo) {
            // -1 表示没有计算过
            Arrays.fill(m, -1); 
        }
        return dp(n - 1, 0, memo, nums, k);
    }

    private long dp(int i, int j, long[][] memo, int[] nums, int k) {
        if (i < 0) {
            return 0;
        }
        if (memo[i][j] != -1) {
            return memo[i][j];
        }
        // nums[i] 增大
        long res = dp(i - 1, 0, memo, nums, k) + Math.max(k - nums[i], 0);
        // nums[i] 不增大
        if (j < 2) {
            res = Math.min(res, dp(i - 1, j + 1, memo, nums, k));
        }
        return memo[i][j] = res; 
    }
```







思路二：给每个数就加上k - nums[i], 那么当前i位置的最小增量数是前三个数的最小增量数加上当前的k-nums[i]。举个例子，[2,3,0,0,2,0]，k = 4, 当下标是3的时候，0 需要加上k - 0才能保证以0为开始的三个元素有等于k的数，在加上[2,3,0]里面最小增量数是1，就是3加上了1等于4, 所以i=3开始的最小增量数是4+1=5。整个数组遍历完之后，最后三个元素的最小增量数里面再取最小的增量数，就是整个数组的最小增量数，这里也相当于是处理了如果元素只有三个的情况。

状态转移方程：dp[i] = min(dp[i - 1], dp[i-2], dp[i-3]) + max(k-nums[i], 0);

初始值dp[0],dp[1],dp[2]要先算好。因为这里当前元素i，只依赖于前三个元素也可以将dp数组改为三个long的变量也可以。

```
 public long minIncrementOperations(int[] nums, int k) {
        long[] dp = new long[nums.length];
        int n = nums.length;
        for (int i = 0 ; i < nums.length; i++) {
            if (i < 3) {
            // 前三个元素
                dp[i] = Math.max(k - nums[i], 0);
            } else {
                dp[i] =  Math.min(Math.min(dp[i-1], dp[i-2]), dp[i-3]) + Math.max(0, k - nums[i]);
            }
        }
        return Math.min(Math.min(dp[n - 1], dp[n-2]), dp[n-3]);

    }
```

