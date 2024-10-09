---
title: 丢失区间
date: 2023/4/11 22:57:00
comments: true
no_word_count: true 
reward: true 
copyright: true 
categories: 
- 每日一题
tags:
- 数组
typora-root-url: ../../../dongyifeng/source/
index_img: /images/index_img/missing-range.jpg
---



> 给定一个已排序的整数数组，其中元素的取值范围为 【lower，upper】（包括边界），返回其缺少的范围。



**样例1**

```python
输入：
nums = [0, 1, 3, 50, 75], lower = 0 and upper = 99
输出：
["2", "4->49", "51->74", "76->99"]
解释：
在区间[0,99]中，缺失的区间有：[2,2]，[4,49]，[51,74]和[76,99]
```



**样例2**

```python
输入：
nums = [0, 1, 2, 3, 7], lower = 0 and upper = 7
输出：
["4->6"]
解释：
在区间[0,7]中，缺失的区间有：[4,6]
```



**思路：**

1. 处理数组两个端点的缺失区间。
   - if nums[0] < lower：表示有缺失缺失区间。
   - if nums[-1] < upper：表示有缺失区间。
2. 处理数组中间数据，如果相邻两个数据差值大于 1，那么这两个数之间有缺失区间。



```python
def get_range(lower, upper):
    if lower == upper: return str(lower)
    return str(lower) + "->" + str(upper)


def find_missing_ranges(nums, lower, upper):
    # write your code here
    if not nums: return [get_range(lower, upper)]
    result = []
    if lower < nums[0]:
        result.append(get_range(lower, nums[0] - 1))

    for i in range(len(nums) - 1):
        if nums[i] + 1 < nums[i + 1]:
            result.append(get_range(nums[i] + 1, nums[i + 1] - 1))
            
    if nums[-1] < upper:
        result.append(get_range(nums[-1] + 1, upper))
    return result
```