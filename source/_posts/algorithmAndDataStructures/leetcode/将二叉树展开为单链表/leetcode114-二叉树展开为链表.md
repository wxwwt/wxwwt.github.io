---
title: leetcode114-二叉树展开为链表
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: 将二叉树展开为单链表
---
### leetcode114 二叉树展开为链表

### 题目：
```
给定一个二叉树，原地将它展开为一个单链表。

 

例如，给定二叉树

    1
   / \
  2   5
 / \   \
3   4   6
将其展开为：

1
 \
  2
   \
    3
     \
      4
       \
        5
         \
          6
```

### 思路：
一开始最初的想法就是遍历一次二叉树，然后把节点依次存入到一个新的链表中。后来写完后发现题目说的是原地展开为一个单链表，那么就是不能创建空间复杂度是大于O(1)的数据结构了（原地算法要求引入的数据结构是常量级别的）。

方法一：
最开始的想法引入了O(n)的栈用来前序遍历二叉树
```
public void flatten(TreeNode root) {
       List<TreeNode> result = new LinkedList<>();
       Stack<TreeNode> stack = new Stack<>();
       TreeNode currNode = root;
       while(currNode != null ||!stack.isEmpty()) {
           while (currNode != null) {
               result.add(currNode);
               stack.push(currNode);
               currNode = currNode.left;
           }
           currNode = stack.pop();
           currNode = currNode.right;
       }

       for (int i = 1; i < result.size(); i++) {
           TreeNode frontNode = result.get(i - 1);
           TreeNode curr = result.get(i);
           // 变成单链表 左子树都是null
           frontNode.left = null;
           frontNode.right = curr;
       }
   }
```
时间复杂度：O(n)
空间复杂度：O(n)


方法二：
使用了常量级的参数来实现了新的算法,方式非常的巧妙。我没想出来，也是看了题解之后才想明白。
看下图帮助理解一下：
如果一个节点的左子树是空的，那么对于这个节点，直接就是符合单链表的结构。
如果一个节点的左子树不是空的，假设节点是1，那么左子树的最后一个节点4，变成单链表后，该节点1的右子树，5->6是挂在4下面的。
因此只要找到左子树中的最后一个节点，把右子树挂在左子树最后一个节点上就可以了，然后将节点1的左子树置为null。再节点往下走，不断的重复该步骤即可。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E7%AE%97%E6%B3%95%E4%B8%8E%E6%95%B0%E6%8D%AE%E7%BB%93%E6%9E%84/%E8%8A%82%E7%82%B9%E5%9B%BE%E5%83%8F.png)
```
public void flatten2(TreeNode root) {
       while (root != null) {
           // 节点的左子树不为null
           if (root.left != null) {
               TreeNode left = root.left;
               TreeNode temp = left;
               // 找到左子树的最后一个元素
               while (temp.right != null) {
                   temp = temp.right;
               }
               temp.right = root.right;
               root.left = null;
               root.right = left;
           }
           // 左子树为null 直接把右子树挂在当前节点上
           root = root.right;
       }
   }
```
时间复杂度：O(n)
空间复杂度：O(1)
