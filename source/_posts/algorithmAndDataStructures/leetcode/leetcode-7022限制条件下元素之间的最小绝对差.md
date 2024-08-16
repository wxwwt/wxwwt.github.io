---
title: leetcode-7022限制条件下元素之间的最小绝对差
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: leetcode
---
leetcode-7022

这道题是358周赛的第三题，当时没做出来，后来弄明白了，记录一下。



#题目

[限制条件下元素之间的最小绝对差](https://leetcode.cn/problems/minimum-absolute-difference-between-elements-with-constraint/)

给你一个下标从 **0** 开始的整数数组 `nums` 和一个整数 `x` 。

请你找到数组中下标距离至少为 `x` 的两个元素的 **差值绝对值** 的 **最小值** 。

换言之，请你找到两个下标 `i` 和 `j` ，满足 `abs(i - j) >= x` 且 `abs(nums[i] - nums[j])` 的值最小。

请你返回一个整数，表示下标距离至少为 `x` 的两个元素之间的差值绝对值的 **最小值** 。

 

**示例 1：**

```
输入：nums = [4,3,2,4], x = 2
输出：0
解释：我们选择 nums[0] = 4 和 nums[3] = 4 。
它们下标距离满足至少为 2 ，差值绝对值为最小值 0 。
0 是最优解。
```

**示例 2：**

```
输入：nums = [5,3,2,10,15], x = 1
输出：1
解释：我们选择 nums[1] = 3 和 nums[2] = 2 。
它们下标距离满足至少为 1 ，差值绝对值为最小值 1 。
1 是最优解。
```

**示例 3：**

```
输入：nums = [1,2,3,4], x = 3
输出：3
解释：我们选择 nums[0] = 1 和 nums[3] = 4 。
它们下标距离满足至少为 3 ，差值绝对值为最小值 3 。
3 是最优解。
```

 

**提示：**

- `1 <= nums.length <= 105`
- `1 <= nums[i] <= 109`
- `0 <= x < nums.length`



# 思路

1.一开始想到的就是双指针，不过是暴力的双指针，想法就是从头指针i=0开始，尾指针从e = i + x开始，一个个遍历过去，计算每一个数字和当前头指针的数字的差的绝对值，然后记录一个min来获取到最小的那个绝对值。因为绝对值是大于等于0的，一旦遍历的时候，min等于0了，就直接返回，不用继续往下遍历了。不过这个方法的时间复杂度是o(n^2)，计算出来超时了，比赛的时候没有想到其他方法。

```java
 public int minAbsoluteDifference(List<Integer> nums, int x) {
        int min = Integer.MAX_VALUE;
        for (int i = nums.size() - 1; i > 0; i--) {
            int e = i - x;
            while (e >= 0) {
                min = Math.min(min, Math.abs(nums.get(i) - nums.get(e)));
                if (min == 0) {
                    return 0;
                }
                e--;
            }
        }

        return min == Integer.MAX_VALUE ? 0 : min;
    }
```

2.后来看了别人的题解，自己调试了几次，才弄明白。

这里可以用TreeSet来维护当前值能够比较的元素，然后因为TreeSet内部使用红黑树实现的，查找的时候时间复杂度是o(logn)，n个元素就是o(nlogn)。

下面在讲一下思路，首先TreeSet是用来维护当前元素能够匹配到的元素，也就是距离当前下标i至少有x距离的元素。因此for循环遍历的i是从x开始的，然后把当前元素左边距离i大于等于x的元素放到set中。

这里小伙伴可能有疑问，为什么不放右边的，只用放左边的？举个栗子，[5,3,2,10,15] , x = 1 , 我们是从i = 1， 数字3开始遍历的，先会左边把满足要求的5放到set，对于右边的2,10,15也是满足距离|i - j| > x，但是不用放，因为当我们i遍历到2的时候，2也会去找左边满足条件的元素，这个时候，2就会把3给放到set中，所以我们不需要处理右边的元素，只要每次把当前元素左边满足条件的元素给放到set中就可以了。

然后我们在调用TreeSet的函数ceiling()，floor()找到比当前元素大和小的元素，找到它们中和当前元素nums[i]差值的绝对值的最小值就可以了。整个数组遍历完，就找到了最小值。

时间复杂度：o(nlogn)

空间复杂度：o(n)

```
  public int minAbsoluteDifference(List<Integer> nums, int x) {
        int n = nums.size();
        int result = Integer.MAX_VALUE;
        TreeSet<Integer> set = new TreeSet<>();
        for (int i = x ; i < n; i++) {
            set.add(nums.get(i - x));
            Integer ceiling = set.ceiling(nums.get(i));
            if (ceiling != null) {
                result = Math.min(result, Math.abs(nums.get(i) - ceiling));
            }

            Integer floor = set.floor(nums.get(i));
            if (floor != null) {
                result = Math.min(result, Math.abs(nums.get(i) - floor));
            }

        }
        return result;
    }
```