---
title: leetcode53_最大子序和
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: leetcode
---
### leetcode53 最大子序和
给定一个整数数组 nums ，找到一个具有最大和的连续子数组（子数组最少包含一个元素），返回其最大和。

示例:

输入: [-2,1,-3,4,-1,2,1,-5,4],
输出: 6
解释: 连续子数组 [4,-1,2,1] 的和最大，为 6。

进阶:
如果你已经实现复杂度为 O(n) 的解法，尝试使用更为精妙的分治法求解。

题解:

解法一:动态规划
首先读完题目我们知道数组最少是一个元素,那么此时它的最大和就是这个元素,不管它是正负整数还是零.那么当变成两个元素的时候呢,我们分析一下:
1.如果第二个元素是正整数,那一定会让最大和变大
2.如果第二个元素是零的话,对最大和其实没有影响
3.如果第二个元素是负数的话,那么最大和就是第一个元素和第二个元素中最大的那个负数,
令dp[i]是数组nums的最大和
推广到i个数的话
1.如果dp[i - 1]是正数或或者0就加上去
2.如果dp[i -1]是负数的话,就不加入,选第nums[i]为最大和
然后我们根据这个条件列出状态转移方程,这里把零放入到第一种情况中,令dp[i]是数组nums的最大和
$$
dp[i]
\begin{cases}
dp[i-1] + nums[i], &if\ dp[i-1] >= 0\\
nums[i], &if\ dp[i-1] <0 \\
\end{cases}
$$
在根据这个状态转移方程写出代码
```java
public static int maxSubArray(int[] nums) {
       int[] dp = new int[nums.length];
       dp[0] = nums[0];
       for (int i = 1; i < nums.length; i++) {
           if (dp[i - 1] < 0) {
               dp[i] = nums[i];
           } else {
               dp[i] = dp[i - 1] + nums[i];
           }
       }

       int max = dp[0];
       for (int i = 1; i < nums.length; i++) {
           max = Math.max(max, dp[i]);
       }
       return max;
   }
```

解法二:分治法
其实感觉分治法不如动态规划来的清晰,但是题目中说进阶可以使用分治法,参考了网上的资料才理解.
我们来想想.如果一个数组中的最大子序和拆成两个部分
1.最大和在左边
2.最大和在右边
3.最大和处于中间,跨越左右两边,此时最大和是左边数组的后边部分和右边数组的左边部分合起来的值
所以通过子数组来计算出最大值,我们需要知道左数组最大值,右数组最大值,局部的最大值,总和,然后根据这些值,比较出全局的最大和.
```java
public int maxSubArray(int[] nums) {
      return mergeCount(nums,0,nums.length)[2];
  }
  /**
   * @return 片段处理后的数组，依次为：左数组最大值，右数组最大值，局部最大值，总和
   * */
  public int[] mergeCount(int[] nums,int fromIndex,int toIndex){
      int[] result=new int[4];
      if(toIndex-fromIndex!=1){
          int midIndex=(toIndex+fromIndex)>>>1;
          int[] resL=mergeCount(nums,fromIndex,midIndex);
          int[] resR=mergeCount(nums,midIndex,toIndex);
          result[0]=Math.max(resL[0],resL[3]+resR[0]);
          result[1]=Math.max(resR[1],resL[1]+resR[3]);
          result[2]=Math.max(Math.max(resL[2],resR[2]),resL[1]+resR[0]);
          result[3]=resL[3]+resR[3];
          return result;
      }
      Arrays.fill(result,nums[fromIndex]);
      return result;
  }
```
