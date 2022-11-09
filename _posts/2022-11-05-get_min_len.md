---
title: 需要排序的最短子数组长度
tags: 每日一题 数组
typora-root-url: ../../dongyifeng.github.io
---

> 给定一个无序数组 arr，求出需要排序的最短子数组长度。
>
> 【例如】arr=【1,5,3,4,2,6,7】返回 4，因为只有【5,3,4,2】需要排序。



分析：

子数组一定是连续的，所以整个数组可以分为三部分，只有中间一部分是乱序的。我们需要找到乱序部分的起始位置和终止位置。

首先我们分析一下这三部分的特征：因为有序1 和有序3 不需要排序，那么有序1排序后一定在最前边一段（数据最小的一段），同理有序3排在最后一段（数据最大的一段）：有序1 < 乱序2 < 有序3

 那么：

- 有序1 < min( 乱序 2 )
- 有序3 > max( 乱序 2 )

如下图：从右向左求出最小值，最小值左边比自己大的数据（橙色数据），顺序都是有问题的，排序后最小值一定在橙色节点前。所以寻找有问题数据的左边界。

同理：从左向右求出最小值，寻找有问题数据的右边界。

![](/../typora/images/algorithm/screenshot-20221108-125304.png)

步骤：

- 从右向左遍历是降序的过程，用 min 记录 i 右侧的最小值。如果 arr[i] > min（此处乱序），说明 min 的值需要放到 arr[i] 左边，用 index 记录 i 位置。找到最左边乱序的位置记录为：no_min_index
- 从左向右遍历是升序的过程，用 max 记录 i 左侧的最大值。如果 arr[i] < max（此处乱序），说明 max 的值需要放到 arr[i] 右边，用 index 记录 i 位置。找到最右边乱序的位置记录为：no_max_index
- 最终结果：no_max_index - no_max_index + 1。最右边开始乱序的位置 - 最右边开始乱序的位置 + 1就是最小乱序子数组的长度。



![](/images/assets/screenshot-20221107-220534.png)

时间复杂度：O(N)

空间复杂度：O(1)

```python
def get_min_len(arr):
    if not arr: return 0
    min_value = arr[-1]
    no_min_index = -1
    for i in range(len(arr) - 2, -1, -1):
        if arr[i] > min_value:
            no_min_index = i
        else:
            min_value = min(min_value, arr[i])

    if no_min_index == -1: return 0

    max_value = arr[0]
    no_max_index = -1
    for i in range(1, len(arr)):
        if arr[i] < max_value:
            no_max_index = i
        else:
            max_value = max(max_value, arr[i])

    return no_max_index - no_min_index + 1
```

