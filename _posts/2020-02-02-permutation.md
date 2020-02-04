---
title: 全排列
key: 10022
tags: 算法
layout: article
category: blog
comment: true
date: 2020-02-02 23:48:00 +08:00
modify_date: 2020-02-02 23:48:00 +08:00
---

回溯算法经常会结合DFS一起出现，leetcode上一道[46.全排列](https://leetcode-cn.com/problems/permutations/)就是回溯+DFS+剪枝思想的典型应用。

## DFS

这道题最常规的一种解法是DFS，然后通过choose or not choose来剪枝回溯。说的比较糙，如果领会不了建议拿一个case用GDB跟一下下面的代码，自己跟一遍胜过千言万语。

```c++
#include <vector>
#include <iostream>
#include <unordered_map>

using namespace std;

void dfs(vector<int>& nums, vector<vector<int>>& ret, vector<int>& permutation, unordered_map<int, int>& choosed)
{
    for (int i = 0; i<nums.size(); i++) {
        if (permutation.size()==nums.size()) {
            ret.push_back(permutation);
            return;
        }
        if (choosed[nums[i]]) { //已经选过了
            continue;
        }
        permutation.push_back(nums[i]);
        choosed[nums[i]] = 1;
        dfs(nums, ret, permutation, choosed);

        //回溯
        permutation.pop_back();
        choosed[nums[i]] = 0;
    }
}

vector<vector<int>> permute(vector<int>& nums)
{
    vector<vector<int>> ret;
    vector<int> permutation;
    unordered_map<int, int> choosed;
    for (int i = 0; i<nums.size(); i++) {
        choosed[nums[i]] = 0;
    }
    dfs(nums, ret, permutation, choosed);
    return ret;
}

int main()
{
    vector<int> nums = {1, 2, 3};
    vector<vector<int>> ret = permute(nums);
    for (int i = 0; i<ret.size(); i++) {
        for (int j = 0; j<ret[0].size(); j++) {
            cout << ret[i][j] << " ";
        }
        cout << endl;
    }

    return 0;
}
```

## 回溯

还有一种是纯回溯的做法，分别把每个元素交换到第一个位置，然后递归的对后续元素做同样操作，代码如下

```c++
void backtrack(vector<int>& nums, vector<vector<int>>& ret, int begin)
{
    if (begin>=nums.size()) { //递归终止条件
        ret.push_back(nums);
        return;
    }

    for (int i = begin; i<nums.size(); i++) {
        swap(nums[i], nums[begin]);
        backtrack(nums, ret, begin+1); //递归对后续元素做同样的操作

        //回溯回来
        swap(nums[i], nums[begin]);
    }
}

vector<vector<int>> permute(vector<int>& nums)
{
    vector<vector<int>> ret;
    backtrack(nums, ret, 0);
    return ret;
}

int main()
{
    vector<int> nums = {1, 2, 3};
    vector<vector<int>> ret = permute(nums);
    for (int i = 0; i<ret.size(); i++) {
        for (int j = 0; j<ret[0].size(); j++) {
            cout << ret[i][j] << " ";
        }
        cout << endl;
    }

    return 0;
}
```