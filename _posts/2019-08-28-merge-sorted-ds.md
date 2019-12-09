---
title: 合并有序数据结构
key: 10008
tags: 算法
layout: article
category: blog
comment: true
date: 2019-08-28 20:49:00 +08:00
modify_date: 2019-08-31 18:45:00 +08:00
---

有序数据结构的合并是个基础操作，在很多算法中有着应用，比如归并排序先分治再归并，其中的归并操作就是合并有序数组的过程，掌握基本功对于后续复杂算法的理解有着非常重要的帮助

## 题目描述

### easy

- [88. Merge Sorted Array](https://leetcode.com/problems/merge-sorted-array/)
- [21. Merge Two Sorted Lists](https://leetcode.com/problems/merge-two-sorted-lists/)
- [617. Merge Two Binary Trees](https://leetcode.com/problems/merge-two-binary-trees/)

### hard

- [23. Merge k Sorted Lists](https://leetcode.com/problems/merge-k-sorted-lists/solution/)

## 思路分析
### 合并有序数组
```go
func merge(nums1 []int, m int, nums2 []int, n int)  {
    i, j, k := m-1, n-1, m+n-1
    for j >= 0 {
        if i >= 0 && nums1[i] > nums2[j] {
            nums1[k] = nums1[i]
            i = i-1
        } else {
            nums1[k] = nums2[j]
            j = j-1
        }
        k = k-1
    }
}
```
倒序遍历，相继比较nums1和nums2，取大的放到顺序位置
- nums1先遍历完，nums2全部放到剩下的位置
- nums2先遍历完，nums1自动有序，无须任何操作
由此可得循环终止条件是nums的下标大于等于0

时间复杂度O(N)，空间复杂度O(1)

### 合并两个有序链表
```go
/**
 * Definition for singly-linked list.
 * type ListNode struct {
 *     Val int
 *     Next *ListNode
 * }
 */
func mergeTwoLists(l1 *ListNode, l2 *ListNode) *ListNode {
    //判空
    if l1 == nil {
        return l2
    }
    if l2 == nil {
        return l1
    }

    //找到头结点
    var head *ListNode
    if l1.Val < l2.Val {
        head = l1
        l1 = l1.Next
    } else {
        head = l2
        l2 = l2.Next
    }

    result := head //记录头结点
    for l1 != nil && l2 != nil { //遍历
        if l1.Val < l2.Val {
            head.Next = l1
            l1 = l1.Next
        } else {
            head.Next = l2
            l2 = l2.Next
        }
        head = head.Next
    }

    if l1 == nil {
        head.Next = l2
    }
    if l2 == nil {
        head.Next = l1
    }
    return result
}
```
- 先比较l1和l2，选出小的那个作为头结点
- 循环终止条件：l1或l2遍历完
- 连接上未遍历完的链表
时间复杂度O(N)，空间复杂度O(1)

### 合并二叉树
以头结点为研究对象
- t1&t2均存在，t1.Val += t2.Val，分别递归左右子树
- t1、t2只存在一个，取存在的那个作为起始结点 
- t1、t2均不存在，循环终止

由此可得终止条件为 `t1 == nil || t2 == nil`

**递归实现**
```go
/**
 * Definition for a binary tree node.
 * type TreeNode struct {
 *     Val int
 *     Left *TreeNode
 *     Right *TreeNode
 * }
 */
func mergeTrees(t1 *TreeNode, t2 *TreeNode) *TreeNode {
    if t1 == nil {
        return t2
    }
    if t2 == nil {
        return t1
    }

    t1.Val += t2.Val
    t1.Left = mergeTrees(t1.Left, t2.Left)
    t1.Right = mergeTrees(t1.Right, t2.Right)
    return t1
}
```
由于不存在重复计算，所以整体性能还可以，比非递归实现更合适


### 合并k个排序链表

这道题的核心思想是利用堆来维护链表有序的性质，建堆的平均时间复杂度是O(N)，所以本题的时间复杂度是O(N)，空间复杂度是O(N)。我觉得这道题主要考察两个点：

- 堆的应用
- while 循环中边界条件的把握，如循环到堆中最后一个元素以及是先pop还是最后pop

由于Golang不支持泛型，且Golang标准库的堆不如C++写起来方便，所以本题用C++实现

```c++
ListNode* mergeKLists(vector<ListNode*>& lists) {
    priority_queue<ListNode*, vector<ListNode*>, cmp> pq; //小顶堆
    for (int i = 0; i < lists.size(); i++) {
        if (lists[i]) {
            pq.push(lists[i]);
        }
    }
    if (pq.empty()) {
        return nullptr;
    }

    ListNode* ret = pq.top();
    ListNode* iter = ret; //迭代器

    pq.pop();
    while (!pq.empty()) {
        //1.push(iter->next)
        //2.pop掉头结点
        //3.iter->next指向当前头结点，此处，头结点有可能不存在
        //4.iter = iter->next
        if (iter->next) {
            pq.push(iter->next);
        }
        iter->next = pq.top();
        iter = iter->next;
        pq.pop();
    }
    return ret;
}
```