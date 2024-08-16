---
title: 954-leetcode
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: essay
---

[leetcode-954链接](https://leetcode-cn.com/problems/array-of-doubled-pairs/)

方法一：  
找出数组中成对的的二倍数组对，第一反应就是求每个数组的值看，是否有当前值的2倍或者1/2，如果每个值有
2倍数或者1/2的数，那么就返回true。然后这个2倍数或者1/2使用hashmap存一下，来判断。
后来发现这里是不能用1/2, 因为整数的除法会忽略小数，-5 / 2 = -2这样结果就不对了，所以只能用乘法来找。
思路：用一个map储存每个值出现的次数key-数值，value-出现次数，从小的数i开始遍历（所以要把arr排个序），如果map中存在2*i的数值，
2*i的次数就减去i出现的次数。判定的条件是：如果i的次数小于2*i的次数，就直接返回false。举个栗子，[2,,2,4,5]，i =2, 2的次数2是,2 * i = 4,
4的次数是1，那么肯定不能成对了。如果是[2,4,4,8], 4的次数是大于2的次数的这样还是有机会凑成对的。
所以只要遍历完arr的时候，判定条件还是成立的就可以返回true。  

注：  
1.先判断0的次数的奇偶，如果是奇数的直接返回false即可，因为0如果是奇数的话，就不会有2 * 0 = 0 的另外一个0出现，就凑不成对  
2.进行循环的时候，要把数组排列成按绝对值从小到大排列，因为如果是负数的话，绝对值大的在前面，2*i的话是在往比i还小的值在找，是找不到结果的。比如说是-4，-2，2，4 这样，先找-4 * 2 = -8 ，直接找-8就返回false了。所以要按照绝对值升序排列，[-2，2，4，-4]这样从-2开始，-2 * 2 = -4，然后找到-4后，-4的次数减去-2出现的次数

```
public boolean canReorderDoubled(int[] arr) {
       List<Integer> tempList = new ArrayList<>();
       for (int x : arr) {
           tempList.add(x);
       }
       Collections.sort(tempList, Comparator.comparingInt(Math::abs));

       Map<Integer, Integer> map = new HashMap<>();
       tempList.forEach(
               item -> map.put(item, map.getOrDefault(item, 0) + 1)
       );

       // 先判断0次数的奇偶 奇数就直接返回false
       if ((map.getOrDefault(0, 0) & 1) == 1) {
           return false;
       }

       for (int item : tempList) {
           // 如果小的数的次数大于2倍数的次数  就不会成对
           if (map.get(item) > map.getOrDefault(2 * item, 0)) {
               return false;
           }
           map.put(2 * item, map.getOrDefault(2 * item, 0) -map.get( item));
           map.put(item, 0);
       }
       return true;
   }
```
时间复杂度：O(nlogn)，统计数值出现次数是O(n),排序是O(nlogn).  
空间复杂度：O(n), 有n个元素要储存在hashmap中，算上次数,和额外的数组的话就是3n，也是O(n)的空间复杂度。  
这样的执行时间大概在100ms多，仅打败14%的人。。。

然后在方法一的基础上改了两个地方，一个是数组中的值并不需要每个都遍历，如果是重复的值，只需要判断一次，
就是把原来的数组的值去重，然后在  
map.put(2 * item, map.getOrDefault(2 * item, 0) -map.get( item));  
map.put(item, 0);  
这两行代码的时候就可以去掉map.put(item, 0);这一行方法一优化版本：
```
public boolean canReorderDoubled2(int[] arr) {
        List<Integer> tempList = new ArrayList<>();
        for (int x : arr) {
            tempList.add(x);
        }
        Collections.sort(tempList, Comparator.comparingInt(Math::abs));

        Map<Integer, Integer> map = new HashMap<>();
        tempList.forEach(
                item -> map.put(item, map.getOrDefault(item, 0) + 1)
        );

        // 先判断0次数的奇偶 奇数就直接返回false
        if ((map.getOrDefault(0, 0) & 1) == 1) {
            return false;
        }

        for (int item : tempList) {
            // 如果小的数的次数大于2倍数的次数  就不会成对
            if (map.get(item) > map.getOrDefault(2 * item, 0)) {
                return false;
            }
            map.put(2 * item, map.getOrDefault(2 * item, 0) -map.get( item));
            map.put(item, 0);
        }
        return true;
    }
```
这样的结果是20多ms，比之前快了好几倍，不过也只打败62%的人，没有想到更好的方法了。


这个下面的是java里面运行时间第一的代码，执行时间2ms，我看几遍也没看懂，贴一下代码瞻仰一下大佬
```
class Solution {
    static final int INF = 200010;

    public boolean canReorderDoubled(int[] arr) {
        int min = INF, max = - INF;
        for (int e: arr) {
            min = Math.min(min, e);
            max = Math.max(max, e);
        }
        int[] map = new int[max - min + 1];
        int offset = - min;
        for (int e: arr) {
            ++map[e + offset];
        }
        for (int num = Math.max(1, min); num <= max; ++num) {
            if (map[num + offset] == 0) {
                continue;
            }
            int partner = num << 1;
            if (partner > max || map[partner + offset] < map[num + offset]) {
                return false;
            }
            map[partner + offset] -= map[num + offset];
        }
        for (int num = Math.min(-1, max); num >= min; --num) {
            if (map[num + offset] == 0) {
                continue;
            }
            int partner = num << 1;
            if (partner < min || map[partner + offset] < map[num + offset]) {
                return false;
            }
            map[partner + offset] -= map[num + offset];
        }
        return true;
    }
}
```
