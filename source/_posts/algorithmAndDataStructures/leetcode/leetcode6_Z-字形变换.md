---
title: leetcode6_Z-字形变换
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: leetcode
---
leetcode-6 Z 字形变换

题目:
将一个给定字符串根据给定的行数，以从上往下、从左到右进行 Z 字形排列。

比如输入字符串为 "LEETCODEISHIRING" 行数为 3 时，排列如下：

L   C   I   R
E T O E S I I G
E   D   H   N
之后，你的输出需要从左往右逐行读取，产生出一个新的字符串，比如："LCIRETOESIIGEDHN"。

请你实现这个将字符串进行指定行数变换的函数：

string convert(string s, int numRows);
示例 1:

输入: s = "LEETCODEISHIRING", numRows = 3
输出: "LCIRETOESIIGEDHN"
示例 2:

输入: s = "LEETCODEISHIRING", numRows = 4
输出: "LDREOEIIECIHNTSG"
解释:

L     D     R
E   O E   I I
E C   I H   N
T     S     G

思路:
思路一: 一开始最初的想法就是构造一个二维数据然后按照题目的顺序将字符存放进去,最后在按照行数组拼接起来输出,构造完以后发现构建这个二维的Z字浪费了很多时间,题目可以做出来就是比较耗时
思路二:
第二种思路是直接从源字符串中找到每一个属于各行数组的字符,然后拼接输出
咱们用数字下标来代替字符看下每行的字符有什么规律
3行:
```
0   4
1 3 5
2   6
```

4行:
```
0     6
1   5 7
2 4   8
3     9
```

5行:
```
0       8
1     7 9
2   6   10
3 5     11
4       12(第4行第1列)
```
我们可以发现下标i和总行数n,行号r([0~n]),间距step = 2n -2,之间的规律大致是这样,
1.第一行的数组下标和最后一个行数组下标相差是n
2.对于间距最大的行的数据规律,行间距最大是2n-2,以3行为例,第一行的数据就是0,4,8,12,每个下标值间距都是step = 2n -2
3.中间行的间距,以5行为例第二行1,7,9间隔是6,2,相加等于8;第三行2,6,10,间距是4,4,相加等于8;
step-2*r
```
// 摘自网上的算法 我自己写的没有这个效率高
public String convert(String s, int numRows) {
		if (numRows == 1) return s;

		int step = numRows * 2 - 2; // 间距
		int index = 0;// 记录s的下标
		int len = s.length();
		int add = 0; // 真实的间距
		string ret;
		for (int i = 0; i < numRows; i++) // i表示行号
		{
			index = i;
			add = i * 2;
			while (index < len)//超出字符串长度计算下一层
			{
				ret += s[index]; // 当前行的第一个字母
				add = step - add;// 第一次间距是step -2*i，第二次是2*i,
				index += (i == 0 || i == numRows-1) ? step : add; // 0行和最后一行使用step间距，其余使用add间距
			}
		}
		return ret;
	}
```
思路三:
第三种想法和第一种有点像,不过不是一次构造整个Z字型的数据,而是先创建出总行数n个数组,然后遍历字符串s,一次往每个数组里面放入字符,
以acbcdefg,3行为例
```
a   e
b d f
c   g
```
开始遍历字符串s
a放入到第一个数组  str[0] = a
b放入到第二个数组  str[1] = b
c放入到第三个数组  str[2] = c
d放入到第二个数组  str[1] = bd
e放入到第一个数组  str[0] = ae
f放入到第二个数组  str[1] = bdf
g放入到第三个数组  str[2] = cg

关键就在于遍历s的时候选择第几个数组去放入,从上往下的时候数组下标是递增的,从下斜着往上是递减的,开始换向是n-1行的字符和下标等于0的字符.

```
public static String convert(String s, int numRows) {
       if (numRows == 1) {
           return s;
       }
       List<StringBuilder> resultList = new ArrayList<>();
       for (int i = 0; i < numRows; i++) {
           resultList.add(new StringBuilder());
       }
       int listIndex = 0, direction = 1;
      // direction 默认是正向就是加1 如果换向就变成-1
       for (int i = 0; i < s.length(); i++) {
           resultList.get(listIndex).append(s.charAt(i));
           if (listIndex == numRows - 1 || (i != 0 && listIndex == 0)) {
               direction *= -1;
           }
           listIndex += direction;
       }

       for (int i = 0; i < resultList.size(); i++) {
           if (i != 0) {
               resultList.get(0).append(resultList.get(i));
           }
       }
       return resultList.get(0).toString();
   }
```
