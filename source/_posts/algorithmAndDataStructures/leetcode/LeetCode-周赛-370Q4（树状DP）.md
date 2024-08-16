---
title: LeetCode-周赛-370Q4（树状DP）
date: 2023-11-05 21:31:36
updated: 2024-08-15 22:35:58
tags: leetcode
---
LeetCode-周赛-370Q4

Q4：[100112. 平衡子序列的最大和](https://leetcode.cn/problems/maximum-balanced-subsequence-sum/description/)


解题思路：

这道题对我来说挺难的，写了好久也没有写出来，一开始想法是直接用dp，dp[i]是以nums[i]结尾的最大元素和，但是写的时候发现dp[i]= max(dp[j] + nums[i], nums[i]), dp[j]是前面的最大元素和，所以得知道dp[j]是多少，但是没想出来怎么算这个值，只能把前面的和都存下来，都比一遍求最大值，所以用两层for循环，但是-10^9 <= nums[i] <= 10^9会超时。后来看了题解知道树状数组用树状数组来处理这样大数据量下的查询，更新才行，这道题其实要考察的东西挺多的。

知识点一：转换
nums[ij] - nums[ij-1] >= ij - ij-1要转换为nums[i] -i >= nums[j] - j, 先把原来题目里面给的判断公式转换一下，这样可以把nums[i] - i 看成一个整体，更方便计算

知识点二：离散化
想用树状数组来计算的前面是数值要都是正整数，而这个题目的数值是-10^9 <= nums[i] <= 10^9，可以看到是有负数的，所以要用离散化先把每个元素都转化为正整数，处理过程是先对原来的元素排序，然后根据排序的下标来当做的新数组的元素并加1（把0变成1，从1开始），这样数值就都会使正数了方便使用树状数组。

知识点三：树状数组
当需要维护前缀和的元素很大的时候可以使用线段树或者树状数组
注意一下：线段树能解决的问题更加广泛，树状数组能解决的，线段树也可以解决，反过来是不行的。下次再专门出两篇介绍线段树和树状数组的文章，这里我们先找几个模板用一下就行，知道树状数组可以用原数组arr的n长度的新数组arr来储存arr的前x个元素的和，max/min等关系就行，而且它的下标是二进制的。

有了上面的几个知识点其实就好做了，递推公式其实不复杂，dp[i] = dp[j] + nums[i], 我们用树状数组来储存了dp[j] (j < i)的最大元素和，查询树状数组里面的元素，更新元素，都是nlogn的时间复杂度，这样即使是10^9也不会超时了。

```
    public long maxBalancedSubsequenceSum(int[] nums) {
        int n = nums.length;
        long ans = Long.MIN_VALUE;
        long[] tmp = new long[n];
        for(int i = 0;i < n;++i){
            tmp[i] = nums[i] - i;
        }
        // 计算值排序
        Arrays.sort(tmp);
        BinIndexTree bit = new BinIndexTree(n);
        for(int i = 0;i < n;++i){
            // 排序+二分对nums[i]-i做离散化处理
            int idx = lower_bound(tmp, nums[i]-i) + 1;
            long dp = bit.query(idx) + nums[i]; 
            bit.update(idx, dp);   
            ans = Math.max(dp, ans);
        }
        return ans;
    }

	 /**
     * 二分求离散值的下标
     * @param arr
     * @param target
     * @return
     */
    public int lower_bound(long[] arr, long target){
        int l = 0, r = arr.length;
        while(l <= r){
            int mid = (l+r)>>1;
            if(arr[mid] >= target) r = mid - 1;
            else l = mid + 1;
        }
        return l;
    }


 /**
     * 树状数组的模板
     */
class BinIndexTree {
    public int n;
    public long[] tree;

    BinIndexTree(int _n) {
        this.n = _n;
        this.tree = new long[n + 1];
    }

    static int lowbit(int x) {
        return x & (-x);
    }

    public long query(int x) {
        long ret = 0;
        while (x > 0) {
            ret = Math.max(ret, tree[x]);
            x -= lowbit(x);
        }
        return ret;
    }

    public void update(int x, long val) {
        while (x <= n) {
            tree[x] = Math.max(tree[x], val);
            x += lowbit(x);
        }
    }

}
```

