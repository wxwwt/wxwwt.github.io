---
title: 环形链表II
date: 2023-11-05 16:13:03
updated: 2024-08-15 22:35:58
tags: 环形链表II
---
### 环形链表II

这是leetcode的142题，朋友说这是链表中比较经典的快慢指针题目，就去做了一下，不过没用快慢指针做出来。
用的hash写的，毕竟hash写法比较容易。但是快慢指针的方式很巧妙，就像用两个已知速度的运动员去测量一个跑道是否是环形的。
用现实的例子听起来很蠢。。。不过方法是听聪明的，待我慢慢说来。呆！先看题

题目：
给定一个链表，返回链表开始入环的第一个节点。 如果链表无环，则返回 null。

为了表示给定链表中的环，我们使用整数 pos 来表示链表尾连接到链表中的位置（索引从 0 开始）。 如果 pos 是 -1，则在该链表中没有环。

说明：不允许修改给定的链表。

 

示例 1：

输入：head = [3,2,0,-4], pos = 1
输出：tail connects to node index 1
解释：链表中有一个环，其尾部连接到第二个节点。


示例 2：

输入：head = [1,2], pos = 0
输出：tail connects to node index 0
解释：链表中有一个环，其尾部连接到第一个节点。


示例 3：

输入：head = [1], pos = -1
输出：no cycle
解释：链表中没有环。


 

进阶：
你是否可以不用额外空间解决此题？

题解：
官方的题解也是两个，一个hash一个叫Floyd算法。
hash：
首先一个链表有没有成环，其实很自然会想到去遍历一遍这个链表，如果是直线的结构，那肯定最后一个元素会指向null嘛。
如果成环的话，遍历就不会停下，一直循环。当然我们肯定不能让程序一直遍历下去，不停止，当发现这个元素不是遍历过了吗？
那就停止，返回“这个链表成精了啊”！（成环）
这里我一开始想的是用一个hashMap的value去保存这个链表的节点，哈哈哈，当然是不行的，java中的hashMap是key做hash的，value是可以重复的。
如果链表的值没有重复的话，那应该还可以，但是链表的节点值是可以重复的，所以用hashMap的key来保存节点就可以了。这里当然用hashSet也是可以的，
理论上说，用set是更正确的用法，毕竟只要存一个节点值就好了，并不需要map一样的键值对。但是因为java中的hashSet底层其实就是用hashMap实现的，
，所以就没用set，我试过了两种方式结果几乎一样，没有什么很大的差距，都只打败了1/3的对手。（这里对于java，用hashMap，hashSet都可以，对于其他语言可能set更合适，其实我也不知道其他语言是怎么实现这俩货，不是重点，略过吧，2333）
```java
public ListNode detectCycle(ListNode head) {
      ListNode currentNode = head;
       Map<ListNode, ListNode> hash = new HashMap<>();
       // 这里有个边界条件就是head可能本来就是空的 要直接返回null
       while (currentNode != null) {
           // 在对链表进行遍历 有出现两次相同 指向节点就是入环的首节点
           // 这里要注意出现两次指的是hash或者说是内存地址 一开始我用的val来判断的出现两次会有问题
           // 因为val的值是可以重复的
           if (hash.containsKey(currentNode)) {
               return currentNode;
           }
           hash.put(currentNode, currentNode);
           currentNode = currentNode.next;
       }
       return null;
   }
```
时间复杂度：o(n)
空间复杂度：o(n)

快慢指针：
官方把这个叫Floyd算法，概念和原理可以去网上搜索下，这里我觉得还是叫快慢指针会更好理解。
作为题目的进阶，要求不使用额外空间，讲道理也不够严谨，实际上两个指针不还是用了空间嘛，
只是说是常数级的空间，很小。费曼学习法，简单来说用最简单的语言把一个事物，概念讲清楚，能让一个小孩子也听懂，
所以咱们用一个跑步的例子来解释，现在有一个跑道，你不知道它是圆形的还是直线的，你现在有两个已知速度的运动员，和一个秒表，
怎么得出这个跑道是圆的还是直线的（你看这个人他又长又高，就像这个道又大又圆，你们来这里跑步很开心，就像我给你们讲解一样很开心，嘿~ 咳咳，一不小心唱起来了）
![]()
咱们假设有两个远动员小红，小蓝，小蓝跑的比小红快，那么他们两只要沿着跑道去跑，如果这个跑道是直的，他们只会在开始跑的时候相遇，
之后小蓝会一直领先小红。但是这个跑道如果是圆的，小蓝小红一直跑着，过了一定的时间，小蓝会再次遇到小红，因为他速度快嘛，小红会反复的被小蓝超越，不过只要除了起点外，相遇一次
就可以知道这个跑道是圆的了。现在假设小蓝的速度是小红的两倍，咱们上图。
我们计算下从开始到相遇的路程，速度小蓝是小红的两倍，时间相同，根据物理公式 路程 = 时间 x 速度。所以小蓝和小红相遇的时候，时间相同，所以小蓝路程是小红的两倍。
我们看下小红的路线图，有黑色向下箭头处是他们相遇的地点。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E7%8E%AF%E5%BD%A2%E9%93%BE%E8%A1%A8II/%E5%B0%8F%E7%BA%A2%E8%B7%91%E7%9A%84%E8%B7%AF%E7%A8%8B.png)
小蓝的
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E7%8E%AF%E5%BD%A2%E9%93%BE%E8%A1%A8II/%E5%B0%8F%E8%93%9D%E8%B7%AF%E7%A8%8B.png)
在根据位置来设定一下路程的变量定义，x，y，z。
![](http://wxwwt-oss.oss-cn-hangzhou.aliyuncs.com/article_picture/%E7%8E%AF%E5%BD%A2%E9%93%BE%E8%A1%A8II/%E8%B7%AF%E7%A8%8B%E5%85%B3%E7%B3%BB.png)

小红的路程是 x + y
小蓝的路程是 x + y + z + y
然后根据上文路程的两倍关系所以 2 * 小红路程 = 小蓝路程
2x + 2y = x + 2y + z。
所以看出来了吗，x = z。惊不惊喜！？意不意外！？只是图上看起来不太相等，实际上是相等的。
我们就知道之前的直线距离x和z是相等的。因此，当两人相遇的时候，在黑色箭头处，我们马上把小红瞬移到开始跑步的起点，
然后在复制一个小红从黑色箭头处开始跑步，因为速度一样，x和z也相同，所以他们会在x和y相交处相遇，这里就是跑道的环形起点。
也就是题目中要找的那个入环的起始节点。
然后代码如下：
```java
public ListNode getFirstMeetNode(ListNode head) {
      ListNode fast = head;
      ListNode slow = head;
      // 两个指针在列表上遍历  相遇就返回这个相遇的节点
      // 没有相遇就是无环
      while (null != slow && null != slow.next) {
          fast = fast.next;
          slow = slow.next.next;
          if (fast == slow) {
              return fast;
          }
      }
      return null;
  }

  public ListNode detectCycle(ListNode head) {
      if (null == head) {
          return null;
      }


      ListNode meetNode = getFirstMeetNode(head);
      if (null == meetNode) {
          return null;
      }

      // 从头开始同速度一起跑一次 相遇就是入环节点
      ListNode point1 = head;
      ListNode point2 = meetNode;
      while (point1 != point2) {
          point1 = point1.next;
          point2 = point2.next;
      }
      return point2;
  }

```
换这个写法马上就打败100%的java提交者了，哈哈哈（所以好的思路的重要性啊）
时间复杂度：o(n)  实际上加起来遍历了两次，不过还是o(n)
空间复杂度：o(1)  就几个临时用的指针。
